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

## Geofence management

Open **Tracking > Geofences** to list and inspect Tenant-scoped polygon geofences. `GEOFENCE_VIEW` permits definition and stable-membership reads, `GEOFENCE_MANAGE` adds create/edit/lifecycle controls, and `GEOFENCE_EVENT_VIEW` reveals transition and unauthorized-transition history. These permissions are independent and the backend remains authoritative.

Create or edit an ordered open ring using explicit Longitude and Latitude fields. Use Add Vertex, Move Up, Move Down, and Remove; every operation is keyboard accessible and the local SVG preview does not change stored coordinates. DEPOT and CUSTOMER_SITE require the logical Organization location UUID. UNAUTHORIZED_ZONE prohibits a location and always keeps entry alerts enabled. The current frontend has no reusable Organization location selector, so location UUID entry is the documented interim convention.

New definitions are DRAFT. DRAFT and DISABLED definitions can be edited or activated; ACTIVE definitions can be disabled with a required reason; DISABLED definitions can be reactivated or retired; retirement requires a reason and is terminal. A stale-version response asks the operator to reload rather than silently overwrite concurrent changes.

Create a DRAFT with a unique name, type, 3–100 WGS84 polygon vertices and entry/exit alert choices. DEPOT and CUSTOMER_SITE require an active same-Tenant Organization location ID; UNAUTHORIZED_ZONE must omit it. Supply `Idempotency-Key` on create and lifecycle commands. Update is allowed only in DRAFT or DISABLED and requires the current `expectedVersion`. Activate only after validation; no Tenant may exceed 500 ACTIVE definitions. Disable and retire require the current version and a reason. Reactivate a DISABLED definition with Activate. Retirement is permanent; there is no delete or reopen action.

Definition and membership lists are bounded to 100 rows per page. Memberships expose confirmed stable state only. Transition history is cursor-bounded to 100 and may be filtered by geofence, Vehicle, type and source-time range; unauthorized history contains only unauthorized-zone entries. Cross-Tenant identifiers appear not found. History responses and audit records do not expose coordinates, provider/device credentials, Driver PII or Customer data.

## Speed monitoring API

The backend operator API is available under `/api/v1/tracking/speed-monitoring`; the dedicated web interface
is scheduled separately. `SPEED_MONITOR_VIEW` reads configured operational rules and current Vehicle states,
`SPEED_MONITOR_MANAGE` creates/updates rules and executes explicit activate, disable and retire commands, and
`SPEED_EVENT_VIEW` reads speeding episode history/detail. These permissions are independent.

Create either a Tenant fallback rule or a route/version rule with a positive threshold no greater than
400 km/h. New rules are DRAFT. DRAFT and DISABLED rules may be edited or activated; ACTIVE rules may be
disabled; DRAFT or DISABLED rules may be retired permanently. Supply `Idempotency-Key` for create and
lifecycle commands, the current `expectedVersion` for update/lifecycle commands, and a reason for disable or
retire. Concurrent or stale changes return a safe conflict and should be reloaded.

Rules and states are paged with a maximum of 100. Episode history requires a UTC `from`/`to` range no greater
than 31 days and allows at most 500 results per cursor page. Thresholds are configured operational controls,
not authoritative legal road limits. UNKNOWN and CONFIGURATION_UNAVAILABLE are truthful states; missing speed
is never displayed as zero. Cross-Tenant identifiers appear not found, and responses exclude coordinates,
raw telemetry, device/provider details, credentials and personal data.

When two eligible consecutive observations confirm a speeding episode, the system creates one durable
same-Tenant IN_APP notification for eligible Dispatchers. Continued observations within the same episode do
not create one notification per packet. A later repeat episode inside the configured ten-minute repeat window
is presented with higher Notification severity. Durable replay does not create a second logical notification.
Notification text describes the configured operational threshold and must not be interpreted as a legal-limit,
violation, disciplinary, licence or payroll decision.

## Known limitations

US-49 geofence management, Notification integration and operator UI are available and independently accepted. Speed detection, management/query API, Dispatcher IN_APP Notification integration and the dedicated rule/state/episode operator frontend are implemented. Idle detection, route deviation, journey replay, the full tracking dashboard, GPS exception workflows and customer location exposure are not available. Automatic retention purge is disabled; configuring a retention policy enables too-old rejection/metadata but does not schedule purge. US-48 and final US-50 fidelity acceptance still require a physical GPS device and verified real provider speed payload.

## Monitor speed in the operator UI

Users with `SPEED_MONITOR_VIEW` can open **Tracking → Speed Monitoring** to review Tenant or route/version
rules and current Vehicle states. Users with `SPEED_MONITOR_MANAGE` can create and edit draft or disabled rules,
activate, disable, reactivate, and permanently retire them. Thresholds are entered explicitly in km/h and must be
greater than zero and no more than 400. The configured values are operational thresholds; the application does
not claim they are live legal road-speed limits. Retired rules cannot be restored, and rules are never deleted.

The **Current states** view shows UNKNOWN, NORMAL or SPEEDING. Configuration unavailable and missing values are
shown explicitly and are never replaced with a false zero. Select a Vehicle row to review the returned rule and
episode references. The UI does not reveal internal candidate samples.

Users with `SPEED_EVENT_VIEW` can open **Tracking → Speed Episodes** independently of rule-view permission.
Choose an ordered UTC interval no longer than 31 days and optionally filter by Vehicle or Driver UUID. History is
loaded with the server cursor. Episode detail shows the frozen threshold source/value, rule version, source-time
evidence, maximum observed speed, exact Tracking WARNING/HIGH severity, repeat count and nullable attribution.
Unknown Driver, Trip or route attribution stays explicitly unknown. No coordinates, raw telemetry, provider or
device credential, Customer identity, inferred Driver identity, discipline action, map or tracking dashboard is
exposed by this workflow.
