# Live Vehicle Tracking Guide

## Prerequisites and access

Open **Live Tracking** from the application navigation. `TRACKING_VIEW` is required for current/last-known positions and device reference summaries; `TRACKING_HISTORY_VIEW` additionally permits bounded history; `TRACKING_DEVICE_MANAGE` permits device registration, lifecycle and Vehicle association actions. Backend Tenant isolation remains authoritative.

## Vehicle positions

The Vehicle positions tab refreshes every 15 seconds while the page is visible and online. Each row labels the position as **CURRENT** or **LAST KNOWN**, and shows coordinates, source timestamp/age, accuracy, trust, freshness and connectivity. Stale data is never presented as live. Open history to inspect up to 24 hours ordered newest first; larger windows are rejected.

## Tracking devices

Authorized device managers can register a provider-neutral external device reference in DRAFT, optionally record a hardware serial reference, bind or rebind it to an installed provider connection, activate/disable/retire it, and start/end an effective-dated Vehicle association. One device and one Vehicle may each have only one active association. RETIRED is terminal. View-only users see masked reference values and never see provider credentials.

The backend management API lets a `TRACKING_DEVICE_MANAGE` operator list installed provider types; create, inspect and update same-Tenant provider connections; replace an opaque credential reference; test a connection; activate, disable or retire it; and request bounded discovery only when the installed adapter advertises that capability. New connections start in DRAFT. Test Connection does not create telemetry. Responses show only safe configuration and whether a credential is configured; they never echo its reference or secret. Privileged changes and denied management attempts are recorded with safe audit facts. Flespi endpoints must use HTTPS on port 443 within the official `flespi.io` hostname zone; localhost, private/link-local/metadata targets, credential-bearing URLs and redirects are rejected. A graphical provider/device onboarding workflow is not available until the later frontend change sets.

Telemetry providers authenticate with an opaque provider key and signed raw payload; any legacy Tenant value cannot select the Tenant. Operators can inspect sanitized Tracking health/metrics through the existing secured management surface without device IDs, coordinates, nonces, signatures, credentials, Driver or Customer data.

## Known limitations

US-48 supplies foundational live/last-known facts only. Geofences, speeding, idle detection, route deviation, journey replay, the full tracking dashboard, GPS exception workflows and customer location exposure are not available. Automatic retention purge is disabled; configuring a retention policy enables too-old rejection/metadata but does not schedule purge. Final acceptance still requires a physical GPS device and real provider payload.
