# External Integrations Operator Guide

## Current availability

US-73 is independently accepted and COMPLETE. The current screen supports only a governed outbound JSON file exchange in the controlled sandbox. Acceptance applies to this Integration platform workflow; it does not mean an ERP, accounting, CRM, HRMS, fuel-card, telematics, payment, insurance, DMS/OCR, REST, webhook, or inbound-file ecosystem is connected.

## Prerequisites and permissions

Open **External Integrations** from the authenticated operations workspace. Navigation and actions appear only when the current user has the relevant permission; the server always rechecks permission and Tenant authority.

- `INTEGRATION_VIEW`: list and view configurations.
- `INTEGRATION_MANAGE`: create or edit draft/disabled configuration and mapping versions.
- `INTEGRATION_TEST`: run the safe connection test.
- `INTEGRATION_ACTIVATE`: enable or disable.
- `INTEGRATION_AUDIT_VIEW`: view exchange and attempt history.
- `INTEGRATION_RECONCILE`: reserved; no current action is available.

## Configure and activate

1. Create an Integration with a unique name and choose the supported values `FILE_EXCHANGE`, `FILE_JSON_V1`, `OUTBOUND`, `INTERNAL_OPERATIONAL_NON_SENSITIVE`, and `US73_BOUNDED_V1`.
2. Select a server-provided endpoint alias. Raw filesystem paths, URLs, relative paths, and user-controlled directories are not accepted or displayed.
3. Define the versioned `US73_PLATFORM_PROBE_V1` declarative mapping. Invalid fields, duplicates, incompatible formats, unsupported contracts, scripting, and more than 100 targets are rejected.
4. Save the draft and run **Test connection**. The test writes and deletes a non-business probe atomically. At most five tests per user/configuration are allowed per minute.
5. Enable within 15 minutes of a successful test. A stale test, changed version, invalid mapping, unsupported capability, or unsafe alias prevents activation.

An active configuration is materially read-only. Disable it before editing or changing an opaque credential reference. Editing creates the next immutable mapping version, and reactivation requires a fresh successful test. There is no delete action.

## Monitor exchanges

The details screen shows read-only safe history: exchange status, attempt count/times, abbreviated mapping/payload hashes, safe result code, external correlation when available, and server-generated target filename. It never shows canonical payload, secret value, full credential reference, provider response, or raw filesystem path.

Statuses are `PENDING`, `IN_PROGRESS`, `RETRY_SCHEDULED`, `SUCCEEDED`, and `FAILED`. Retryable I/O failures use at most five total attempts. Permanent failures and exhausted retries end as `FAILED`; there is no manual retry, mark-success, payload edit, or history deletion.

## Disable and safety behavior

Disable stops new claims and attempts while preserving history. Same-event replay is idempotent and does not create a second exchange/file. A pre-existing final file with a different hash fails with `INTEGRATION_FILE_INTEGRITY_FAILURE`. Cross-Tenant identifiers return safe not-found behavior.

## Known limitations

Only non-sensitive platform-probe JSON is supported, with a 32 KiB maximum. No business-domain integration, inbound operation, arbitrary send, manual reconciliation mutation, secret administration, or external provider is available in the current release.
