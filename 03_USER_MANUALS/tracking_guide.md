# Live Vehicle Tracking Guide

## Prerequisites and access

Open **Live Tracking** from the application navigation. `TRACKING_VIEW` is required for current/last-known positions and device reference summaries; `TRACKING_HISTORY_VIEW` additionally permits bounded history; `TRACKING_DEVICE_MANAGE` permits device registration, lifecycle and Vehicle association actions. Backend Tenant isolation remains authoritative.

## Vehicle positions

The Vehicle positions tab refreshes every 15 seconds while the page is visible and online. Each row labels the position as **CURRENT** or **LAST KNOWN**, and shows coordinates, source timestamp/age, accuracy, trust, freshness and connectivity. Stale data is never presented as live. Open history to inspect up to 24 hours ordered newest first; larger windows are rejected.

## Tracking devices

Authorized device managers can register a provider-neutral external device reference in DRAFT and optionally record a hardware serial reference. First create and activate a supported Provider Connection. Then open **Tracking → Devices**, choose **Add Device**, select that connection and save the DRAFT. Open Device details, choose **Bind Provider**, enter or select the provider-side reference according to the installed adapter's advertised capabilities, choose **Associate Vehicle**, and then **Activate Device** explicitly. Activation is rejected until both the ACTIVE binding and current same-Tenant Vehicle association exist. Device details return the current safe binding/association state so a refreshed session can resume and use the current optimistic version for rebind.

An ACTIVE Device can be disabled without deleting historical Tracking data and reactivated when its readiness facts remain valid. **Rebind Provider** reloads current backend detail and uses the current binding version; a concurrent change is rejected and the latest detail is shown instead of overwriting it. **Retire Device** is permanent and removes all mutation actions without hard deletion. One Device and one Vehicle may each have only one active association. View-only users see masked reference values and never receive provider credentials or binding mutation versions. Supported installed adapters allow these connection and Device changes at runtime without application restart or deployment; adding an unsupported provider protocol still requires a reviewed adapter release.

## Provider connections

Operators with `TRACKING_DEVICE_MANAGE` can open **Tracking → Provider Connections**. The page lists same-Tenant connections with provider type and alias, lifecycle, safe test status, credential-configured state, poll interval and safe health timestamps. Open **Details** for endpoint, safe configuration and backend-reported capability tags. Provider credentials and their stored references are never displayed.

Choose **Add Provider Connection**, select one of the installed provider types loaded from the backend, and enter the display name, alias, opaque provider key ID, approved HTTPS endpoint, opaque credential reference, polling interval, page size and bounded safe configuration. Connections are saved as DRAFT; creation never starts polling automatically. Safe configuration accepts an object of string values up to 8 KiB and rejects secret-like keys. The backend remains authoritative for provider-specific fields and SSRF protection; current Flespi endpoints must use HTTPS/443 in the `flespi.io` hostname zone.

Use **Test Connection** to validate configuration and provider access. It reports Connection successful, Authentication failed, Provider unreachable or Configuration invalid without showing raw provider errors. Testing does not activate polling, create a device or ingest telemetry. **Activate** makes an eligible DRAFT/DISABLED connection available to the polling coordinator. **Disable** stops polling until reactivation while preserving historical Tracking data. **Retire** is permanent, stops future polling and cannot be reversed; it is not data deletion. To update safe settings, choose **Edit** and submit the optimistic current version. Leave the replacement credential field blank to preserve the configured credential; the existing reference is never prefilled. Discovery appears only for adapters that advertise `DISCOVERY` and returns a bounded masked preview. FLESPI does not advertise discovery, so its Device workflow requires manual external-reference entry.

Privileged changes and denied management attempts are recorded with safe audit facts. Cross-Tenant identifiers remain not-found-shaped, and view/history-only roles cannot access provider-management APIs or controls.

Telemetry providers authenticate with an opaque provider key and signed raw payload; any legacy Tenant value cannot select the Tenant. Operators can inspect sanitized Tracking health/metrics through the existing secured management surface without device IDs, coordinates, nonces, signatures, credentials, Driver or Customer data.

## Known limitations

US-48 supplies foundational live/last-known facts only. Geofences, speeding, idle detection, route deviation, journey replay, the full tracking dashboard, GPS exception workflows and customer location exposure are not available. Automatic retention purge is disabled; configuring a retention policy enables too-old rejection/metadata but does not schedule purge. Final acceptance still requires a physical GPS device and real provider payload.
