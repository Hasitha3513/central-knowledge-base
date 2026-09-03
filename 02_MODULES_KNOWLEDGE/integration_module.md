# Integration Module

## Ownership and MVP scope

The top-level `integration` bounded context owns provider-neutral external connectivity, declarative mapping, configuration lifecycle, health, external exchange/attempt state, and safe audit facts. US-73 is `COMPLETE` at Flyway V61 after independent technical closure and hostile final acceptance.

Phase 1 implements only `FILE_EXCHANGE / FILE_JSON_V1 / OUTBOUND` through `GOVERNED_OUTBOUND_JSON_FILE_EXCHANGE`, with `US73_PLATFORM_PROBE_V1` as the only registered source contract and `INTERNAL_OPERATIONAL_NON_SENSITIVE` as the only accepted classification. Integration consumes the shared P1-01 durable envelope through handler `integration-outbound-exchange`; it does not own or duplicate `integration_outbox_event`.

Phase 2/Post-MVP includes any ERP, accounting, CRM, HRMS, fuel-card, telematics, payment, insurance, DMS/OCR, REST, webhook, inbound file/API, manual retry, or reconciliation mutation. Each requires independent governance and acceptance.

## Inbound use cases

- List/create/get/update Tenant-owned configurations.
- Run a rate-limited safe connection test; five tests per actor/configuration per minute.
- Enable after a successful current-version test within 15 minutes; disable explicitly.
- List read-only exchange and immutable attempt evidence with default page 20 and maximum 100.
- Accept the registered probe fact durably and idempotently; claim/process no more than 50 due exchanges per Tenant per cycle.

Tenant identity comes from authenticated `CurrentTenant`, never request payloads. Active configurations and active mapping versions are immutable. External delivery is at-least-once, with five attempts, a five-minute claim lease, and no global ordering claim.

## Outbound ports and adapters

- Integration persistence port -> V61 PostgreSQL/JPA adapter.
- Exchange adapter port -> governed outbound JSON filesystem adapter.
- Secret-resolution port -> environment-backed resolver; only opaque references are persisted.
- Audit port -> append-only Integration audit persistence.
- Shared `DurableEventPublisher` -> P1-01 `integration_outbox_event`; Integration creates no second outbox.

The filesystem adapter accepts only server-configured endpoint aliases, writes one UTF-8 JSON document per exchange using `.part` plus atomic rename, rejects unsafe/symlink/traversal paths, enforces 32 KiB, and treats same-hash replay as success and different-hash replay as terminal integrity failure.

## Security and permissions

V61 seeds `INTEGRATION_VIEW`, `INTEGRATION_MANAGE`, `INTEGRATION_TEST`, `INTEGRATION_ACTIVATE`, `INTEGRATION_AUDIT_VIEW`, and reserved `INTEGRATION_RECONCILE`. The reserved permission grants no mutation endpoint. Future financial/restricted classifications require creator/activator segregation, but the accepted non-sensitive probe does not require dual control.

Responses expose only `credentialConfigured` and a masked safe label. Secrets, full credential references, raw paths, canonical payloads, provider bodies, and authorization material are absent from responses/audit/logs.

## Database schema (V61)

#### Table: `integration_configuration`

- **Purpose:** Tenant-owned endpoint configuration, lifecycle, health, current mapping, and safe credential reference.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY; unique with Tenant | Configuration identity |
| `tenant_id` | UUID | NO | - | Tenant scope; leading indexes | Tenant authority |
| `name` / `normalized_name` | VARCHAR(160) | NO | - | normalized value unique per Tenant | Display and business key |
| `integration_type` | VARCHAR(32) | NO | - | `FILE_EXCHANGE` | Capability |
| `protocol` | VARCHAR(32) | NO | - | `FILE_JSON_V1` | Protocol |
| `direction` | VARCHAR(24) | NO | - | `OUTBOUND` | Direction |
| `endpoint_alias` | VARCHAR(80) | NO | - | server allow-list key | Never a raw path |
| `credential_reference` | VARCHAR(160) | YES | NULL | opaque reference only | No secret value |
| `current_mapping_id` | UUID | YES | NULL | Tenant-consistent FK to mapping, RESTRICT | Active mapping |
| `data_classification` | VARCHAR(64) | NO | - | `INTERNAL_OPERATIONAL_NON_SENSITIVE` | Current classification |
| `retry_policy` | VARCHAR(40) | NO | - | `US73_BOUNDED_V1` | Five-attempt policy |
| `lifecycle` | VARCHAR(24) | NO | - | `DRAFT`, `ACTIVE`, `DISABLED` | Lifecycle |
| `health` | VARCHAR(24) | NO | - | allowed health values | Independent health |
| `last_tested_at` / `last_successful_exchange_at` | TIMESTAMPTZ | YES | NULL | - | Operational timestamps |
| `last_tested_version` | BIGINT | YES | NULL | - | Fresh-test binding |
| `version` | BIGINT | NO | 0 | optimistic lock | Concurrency |
| `created_at` / `updated_at` | TIMESTAMPTZ | NO | - | - | Audit time |
| `created_by` / `updated_by` | VARCHAR(255) | NO | - | - | Safe actor identity |

#### Table: `integration_mapping`

- **Purpose:** Immutable declarative mapping versions.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` / `tenant_id` | UUID | NO | - | primary identity; unique pair | Tenant mapping identity |
| `configuration_id` | UUID | NO | - | Tenant-consistent FK, RESTRICT | Owning configuration |
| `mapping_key` | VARCHAR(80) | NO | - | unique with Tenant/config/version | Logical mapping key |
| `mapping_version` | INTEGER | NO | - | positive | Immutable version |
| `source_contract` / `target_schema` | VARCHAR(100) | NO | - | registered allow-list | Contract identities |
| `source_version` / `target_version` | INTEGER | NO | - | positive | Schema versions |
| `rules` | JSONB | NO | - | validated declarative rules | No scripting |
| `definition_hash` | VARCHAR(64) | NO | - | lower-case SHA-256 | Immutable evidence |
| `lifecycle` | VARCHAR(24) | NO | - | `ACTIVE`, `SUPERSEDED` | Mapping state |
| `created_at` / `created_by` | TIMESTAMPTZ / VARCHAR(255) | NO | - | - | Creation audit |

#### Table: `integration_exchange`

- **Purpose:** Durable, idempotent external-delivery state and minimized retry payload.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` / `tenant_id` | UUID | NO | - | primary identity; unique pair | Tenant exchange identity |
| `configuration_id` | UUID | NO | - | Tenant-consistent FK, RESTRICT | Target configuration |
| `source_event_id` | UUID | NO | - | idempotency tuple | Stable event ID |
| `source_event_type` | VARCHAR(100) | NO | - | registered contract | Event type |
| `mapping_version_id` | UUID | NO | - | Tenant-consistent FK, RESTRICT | Mapping snapshot |
| `mapping_definition_hash` / `payload_hash` | VARCHAR(64) | NO | - | lower-case SHA-256 | Integrity evidence |
| `canonical_payload` | JSONB | NO | - | minimized; maximum 32 KiB in application | Retry payload |
| `status` | VARCHAR(24) | NO | - | exact exchange states | Delivery state |
| `attempt_count` | INTEGER | NO | 0 | 0..5 | Attempts consumed |
| `next_attempt_at` | TIMESTAMPTZ | NO | - | Tenant/status/due partial index | Claim eligibility |
| `locked_until` | TIMESTAMPTZ | YES | NULL | stale-claim partial index | Five-minute lease |
| `external_correlation_id` | VARCHAR(160) | YES | NULL | safe metadata | External trace |
| `target_filename` | VARCHAR(80) | YES | NULL | server-generated | Safe filename |
| `last_error_code` | VARCHAR(80) | YES | NULL | sanitized only | Latest safe failure |
| `created_at` / `updated_at` / `completed_at` | TIMESTAMPTZ | mixed | - | history index | Lifecycle times |
| `version` | BIGINT | NO | 0 | optimistic lock | Concurrency |

Unique idempotency key: `(tenant_id, configuration_id, source_event_id, mapping_version_id)`.

#### Table: `integration_exchange_attempt`

- **Purpose:** Immutable attempt chronology.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` / `tenant_id` | UUID | NO | - | primary identity; unique pair | Tenant attempt identity |
| `exchange_id` | UUID | NO | - | Tenant-consistent FK, RESTRICT | Owning exchange |
| `attempt_number` | INTEGER | NO | - | 1..5; unique per Tenant/exchange | Ordered attempt |
| `started_at` / `completed_at` | TIMESTAMPTZ | NO | - | - | Timing |
| `latency_ms` | BIGINT | NO | - | non-negative | Duration |
| `outcome` | VARCHAR(24) | NO | - | success/retryable/permanent | Result |
| `error_code` | VARCHAR(80) | YES | NULL | sanitized only | Safe failure |
| `external_correlation_id` | VARCHAR(160) | YES | NULL | safe metadata | Trace |
| `target_filename` | VARCHAR(80) | YES | NULL | server-generated | Safe target evidence |

#### Table: `integration_audit_event`

- **Purpose:** Append-only privacy-safe Integration action evidence.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` / `tenant_id` | UUID | NO | - | primary/Tenant identity | Audit identity |
| `actor` | VARCHAR(255) | NO | - | - | Safe actor |
| `action` / `target_type` | VARCHAR(64) | NO | - | - | Audited action/target |
| `target_id` | UUID | NO | - | logical target reference | Target identity |
| `outcome` | VARCHAR(24) | NO | - | `SUCCESS`, `FAILURE`, `DENIED` | Result |
| `safe_code` | VARCHAR(80) | YES | NULL | sanitized only | Result code |
| `before_hash` / `after_hash` | VARCHAR(64) | YES | NULL | lower-case SHA-256 if present | Change evidence |
| `correlation_id` | VARCHAR(160) | YES | NULL | safe metadata | Trace |
| `occurred_at` | TIMESTAMPTZ | NO | - | Tenant/target/history index | Event time |

## Verification status

Independent final acceptance passed all three source criteria under the controlled-sandbox evidence tier: focused Integration 24/24; P1-01 plus US-69/70 regression 40/40; Maven 1276/0/0 with 15 skips in 05:04; architecture 44/44; PostgreSQL Flyway V1-V61; real Chromium 6/6 in 1.3 minutes; Vitest 60 files/260 tests; Checkstyle, PMD, SpotBugs, TypeScript, production build, and changed-file lint pass. The 71 global ESLint errors remain confined to eight unchanged Delivery files, with zero US-73-introduced errors. Acceptance means the Integration platform capability is complete; no named ERP/vendor ecosystem is connected. Next task: `US-78-OPERATIONAL-EXCEPTIONS-PRODUCT-DECISIONS-001`.
