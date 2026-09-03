# ADR-US73: External Integration Boundary

- **Status:** Accepted decision; `IMPLEMENTED_ACCEPTANCE_PENDING_US73`
- **Decision task:** `US-73-EXTERNAL-INTEGRATIONS-PRODUCT-DECISIONS-001`
- **Story accounting:** Unchanged at 65 / 87 accepted and 22 / 87 remaining

## Context

US-73 requires administrators to configure supported external exchanges and to see explicit failure, retry, and synchronization state. The source requirements name ERP, accounting, CRM, HRMS, fuel-vendor, telematics, payment, insurance, DMS, API, webhook, and file ecosystems, but do not select a vendor or require all of them to be implemented. P1-01 already owns the shared durable event outbox.

## Decision

1. The dedicated top-level `integration` bounded context is implemented. Integration owns endpoint configuration, declarative mapping, external exchange state/attempts, health, and Integration audit records. It does not own or decide business facts.
2. The only US-73 executable adapter is `GOVERNED_OUTBOUND_JSON_FILE_EXCHANGE`, capability `FILE_EXCHANGE`, protocol `FILE_JSON_V1`, direction `OUTBOUND`. Acceptance uses real, isolated filesystem I/O in a controlled sandbox. Every named vendor ecosystem, generic API, webhook, inbound exchange, bidirectional exchange, and broker is future scope.
3. Configuration is Tenant-owned. Paths are server-managed aliases to allow-listed roots; operators cannot enter raw filesystem paths or network URLs. Configuration states are `DRAFT -> ACTIVE -> DISABLED`. Activation requires a successful non-business probe within the preceding 15 minutes. Active configuration is immutable until disabled. Health is a separate derived state: `UNKNOWN`, `HEALTHY`, `DEGRADED`, `UNAVAILABLE`, or `AUTH_FAILED`.
4. Credentials are opaque references resolved by a provider-neutral `IntegrationSecretResolver`. The accepted file adapter needs none. Secret values are never returned by REST/UI or written to payload, audit, error, or log records.
5. Mapping is declarative and versioned. Allowed operations are field selection, rename, non-secret defaults, typed formatting, and omission of optional null fields, bounded to 100 fields and nesting depth 10. Scripts, templates with code execution, reflection, SpEL, arbitrary expressions, and business-rule mutation are forbidden.
6. US-73 reuses P1-01's `integration_outbox_event` and `DurableEventPublisher`; it creates no second outbox/inbox, broker, or exactly-once claim. The registered consumer name is `integration-outbound-exchange`. Cross-module contracts require separate explicit producer, consumer, payload, version, classification, and idempotency approval.
7. External exchange state is Integration-owned: `PENDING`, `IN_PROGRESS`, `RETRY_SCHEDULED`, `SUCCEEDED`, or `FAILED`. A five-minute claim lease and a maximum batch of 50 per Tenant apply. Delivery is at-least-once and globally unordered. Five total external attempts use immediate, 30-second, 2-minute, 10-minute, and 30-minute scheduling. Permanent validation/security/configuration failures do not retry.
8. Idempotency is unique on `(tenant_id, configuration_id, source_event_id, mapping_version_id)`. Reconciliation is read-only in US-73; there is no manual retry, mark-success, payload edit, or mutation endpoint.
9. Canonical payloads are UTF-8 JSON and at most 32 KiB. The accepted adapter allows only explicitly approved non-sensitive operational data. Future financial, restricted, personal, or secret-bearing contracts require data-classification, encryption/key, and segregation decisions before activation. Safe successful payloads may be retained 30 days and failed payloads 90 days; metadata and audit retention remain governed separately.
10. File delivery writes one file per exchange under the configured allow-listed root, uses an exchange-UUID filename, rejects traversal and symlink escape after canonical-path validation, writes a `.part` file then atomically renames without overwrite, records SHA-256, and treats an existing same-name/same-hash file as idempotent success. A hash mismatch is terminal. Directories use `0750` and files `0640`. Malware scanning is not applicable to system-generated outbound JSON; future inbound files require a new decision.
11. Network SSRF and webhook replay controls are not executable in this adapter because no URL or inbound route exists. Any future HTTPS adapter must use TLS on port 443, forbid redirects, re-resolve DNS at connect time, and deny loopback/private/link-local/metadata destinations. Any future webhook must use a 256-bit hashed endpoint identity, signature verification, a five-minute timestamp window, event-id replay protection, a 32-KiB bound, and rate limiting.
12. Implementation used forward migration V61 after verifying V60 as the prior head. Historical migrations remain unchanged.

## Consequences

- The acceptance proof is deliberately narrow but real: configure, test, activate, exchange through the filesystem, observe retry/failure and audit evidence, prove Tenant/RBAC isolation, disable, and prove no new exchange starts.
- No public webhook, raw-payload read, secret read, manual retry, arbitrary send, delete, or reconciliation mutation API is approved.
- Implementation proceeds in four slices: configuration/security, mapping/probe, durable exchange/retry, and UI/real acceptance. Product acceptance remains pending until implementation, technical closure, and hostile final acceptance complete.
- V61 implements the five Tenant-owned Integration tables and six permissions. Controlled-sandbox evidence passed real PostgreSQL/Flyway V1-V61, real filesystem exchange, 6/6 Chromium scenarios, 1276-test Maven verification, 44/44 architecture checks, and all scoped static/frontend gates. The next task is `US-73-EXTERNAL-INTEGRATIONS-TECHNICAL-CLOSURE-001`; program accounting remains 65/87 accepted.
