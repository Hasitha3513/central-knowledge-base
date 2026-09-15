# Live Vehicle Tracking Guide

## Prerequisites and access

Open **Live Tracking** from the application navigation. `TRACKING_VIEW` is required for current/last-known positions and device reference summaries; `TRACKING_HISTORY_VIEW` additionally permits bounded history; `TRACKING_DEVICE_MANAGE` permits device registration, lifecycle and Vehicle association actions. Backend Tenant isolation remains authoritative.

## Vehicle positions

The Vehicle positions tab refreshes every 15 seconds while the page is visible and online. Each row labels the position as **CURRENT** or **LAST KNOWN**, and shows coordinates, source timestamp/age, accuracy, trust, freshness and connectivity. Stale data is never presented as live. Open history to inspect up to 24 hours ordered newest first; larger windows are rejected.

## Tracking devices

Authorized device managers can register a provider-neutral external device reference in DRAFT and optionally record a hardware serial reference. First create and activate a supported Provider Connection. Then open **Tracking → Devices**, choose **Add Device**, select that connection and save the DRAFT. Open Device details, choose **Bind Provider**, enter or select the provider-side reference according to the installed adapter's advertised capabilities, choose **Associate Vehicle**, and then **Activate Device** explicitly. Activation is rejected until both the ACTIVE binding and current same-Tenant Vehicle association exist. Device details return the current safe binding/association state so a refreshed session can resume and use the current optimistic version for rebind.

An ACTIVE Device can be disabled without deleting historical Tracking data and reactivated when its readiness facts remain valid. **Rebind Provider** reloads current backend detail and uses the current binding version; a concurrent change is rejected and the latest detail is shown instead of overwriting it. **Retire Device** is permanent and removes all mutation actions without hard deletion. One Device and one Vehicle may each have only one active association. View-only users see masked reference values and never receive provider credentials or binding mutation versions. Supported installed adapters allow these connection and Device changes at runtime without application restart or deployment; adding an unsupported provider protocol still requires a reviewed adapter release.

## Journey Replay availability

Journey Replay is available to users with `JOURNEY_REPLAY_VIEW`. Select exactly one Vehicle or Trip and a UTC
range no longer than seven days. Results are chronological, cursor-paged and capped at 1,000 points per page
and 20,000 points per browser session. Playback starts paused and supports keyboard-operable seek and speed
controls. Stop analysis is deterministic and bounded; partial retention, data gaps and quality limitations
remain explicit. Export and engine/idle inference are not available.

Users with `JOURNEY_REPLAY_INCIDENT_VIEW` additionally receive accepted geofence evidence by default and may
opt into producer-labelled Speed or Route Deviation technical evidence. These labels do not upgrade the
producer story's acceptance status. The service permits 30 replay requests per user/minute and 120 per
Tenant/minute on each application instance. A rate-limited request returns a retry instruction; wait at least
60 seconds before trying again. Operations may disable the backend endpoints and matching frontend navigation
with the coordinated journey-replay feature flags without deleting historical telemetry or audit evidence.

Journey Replay is technically complete, but final physical acceptance is on an external-prerequisite hold.
No field cases have passed or failed because the field phase has not started. Final acceptance requires a
genuine provider/device journey retained through the production telemetry path, privacy-safe real stop/gap
and overlay evidence where available, and authorized operator sign-off. Automated or simulated evidence does
not replace that field phase. This hold does not remove or reduce the available technically verified workflow.

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

US-49 geofence management, Notification integration and operator UI are available and independently accepted. Speed detection, management/query API, Dispatcher IN_APP Notification integration and the dedicated rule/state/episode operator frontend are implemented. Route-deviation detection, management/query/review API, Notification integration and operator frontend are implemented. Journey Replay is available under Tracking to users with `JOURNEY_REPLAY_VIEW`: select one Vehicle or Trip, choose an ordered range up to seven days, load the replay, then use play/pause, seek and the `0.5x`, `1x`, `2x`, `4x` or `8x` speed. Playback starts paused. Gaps, partial retention and quality limitations remain visible, stops have a details list, and coordinates appear only after explicit expansion. Users who also hold `JOURNEY_REPLAY_INCIDENT_VIEW` see accepted geofence transitions by default and may opt into speed or route-deviation technical evidence. Those opt-in sources remain visibly labelled as field-fidelity or field-acceptance pending; replay does not upgrade their acceptance. Incident failures can be retried without losing the movement timeline. Idle detection, the full tracking dashboard, GPS exception workflows and customer location exposure are not available. Automatic retention purge is disabled; configuring a retention policy enables too-old rejection/metadata but does not schedule purge. US-48 and final US-50/US-52 field acceptance still require physical evidence.

## Route-deviation API

The backend operator API is available under `/api/v1/tracking/route-deviations`; there is no CS04 frontend
page. `ROUTE_DEVIATION_VIEW` reads rules and current Vehicle state, `ROUTE_DEVIATION_MANAGE` creates/updates
rules and runs explicit lifecycle commands, `ROUTE_DEVIATION_EVENT_VIEW` reads minimized episode/review
history, and `ROUTE_DEVIATION_APPROVE` performs eligible review decisions. These permissions are independent.

Rules accept a 10–5,000 metre inclusive tolerance. New rules are DRAFT. Supply `Idempotency-Key` for create,
update and lifecycle commands, the current `expectedVersion` for mutation, and a reason for disable/retire.
Rule/state lists cap at 100. Episode history requires a source-time window no greater than 31 days and caps
each cursor page at 500. Cross-Tenant identifiers appear not found.

Confirmed deviations create one same-Tenant IN_APP notification for active Dispatchers. WARNING episodes
appear as WARNING; HIGH episodes appear as CRITICAL in the Notification platform. A directly confirmed HIGH
episode does not also create a distance escalation. A later WARNING-to-HIGH transition creates one
`DISTANCE_HIGH` escalation, and rejecting a HIGH episode creates one `REVIEW_REJECTED` escalation. Continued
HIGH telemetry and duplicate durable delivery do not create repeated logical notifications. Notification
content includes the Vehicle, route revision, rounded distance, source time and episode reference, but never
coordinates, route geometry, review notes, provider/device details, credentials, driver identity or personal
data. Email and SMS are not enabled for this workflow.

Only HIGH episodes require review. An authorized reviewer may approve or reject using a governed reason;
`UNKNOWN` requires a 10–500 character note. A correction appends evidence instead of replacing the prior
decision, and a different authorized actor must perform a reversal. Reload after a stale-version conflict.
Responses and audits omit coordinates, raw telemetry, provider/device credentials and personal data. No
route-deviation workflow creates Operations cases.

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

## Monitor route deviations in the operator UI

Users with `ROUTE_DEVIATION_VIEW` can open **Tracking → Route Deviations** to inspect same-Tenant
rules and current Vehicle state. Users with `ROUTE_DEVIATION_MANAGE` can create or update rules and
use the explicit activate, disable and retire actions. A rule tolerance must be from 10 to 5,000 metres;
retired rules cannot be reactivated. UNKNOWN and non-evaluable states remain explicit rather than being
shown as on-route.

Users with `ROUTE_DEVIATION_EVENT_VIEW` can query bounded UTC episode history and open minimized episode
and immutable review detail. The UI shows the assigned route and immutable revision, WARNING/HIGH severity,
rounded deviation distance, effective tolerance and source timestamps. It does not expose coordinates,
route geometry, raw telemetry, provider/device details, credentials, signatures or Driver/Customer PII.

Only HIGH episodes require a review. Users with `ROUTE_DEVIATION_APPROVE` may approve or reject an eligible
episode with the governed reason and expected version. A later correction appends evidence; it never edits
the earlier review, and the reviewer cannot reverse their own decision. Reload after a stale-version conflict.
Confirmed deviations notify active same-Tenant Dispatchers in-app. WARNING maps to WARNING and HIGH maps to
the Notification platform's existing CRITICAL severity. Repeated HIGH telemetry, approval, closure and normal
progress do not create notification floods.

US-52 is technically complete at V92 but is externally blocked for acceptance. Final acceptance requires a separate
physical provider/device journey proving real coordinate and accuracy fidelity on a safe controlled route,
recovery and clearance, Tenant/limited-role denial, privacy, recipient behavior and operator sign-off. Do not
treat US-48 or US-50 physical evidence as US-52 acceptance.

## Provider telemetry delivery

Active provider integrations continue to send signed telemetry to
`POST /api/integration/v1/tracking/positions`. The provider must use its assigned key, current
timestamp, unique nonce and valid HMAC signature; Tenant and Vehicle values in payloads are not
trusted authority. Tracking resolves the provider connection, Device and source-time Vehicle
association from server-side same-Tenant configuration and applies the configured Flespi, Traccar
or Generic normalizer.

A `202 Accepted` response now means Kafka durably acknowledged the canonical normalized record.
Kafka unavailability or acknowledgement timeout returns a safe service-unavailable response, so
operators should restore the broker and retry with the provider's governed idempotency behavior.
Invalid signatures, stale timestamps, replayed nonces, unsupported versions, oversized batches,
unknown Devices and inactive bindings fail closed. Responses and logs never expose credentials,
raw signatures or raw provider payloads.

## Live telemetry cache operations

When the governed hybrid-storage feature flag is enabled, accepted Kafka telemetry is projected to
Redis for current fleet state. TS03 introduces no new operator UI or public API. Each Vehicle's live
projection expires 24 hours after its most recently processed canonical record; stale source records
cannot replace a newer position, while exact replays safely refresh the expiry window.

Operations should monitor the `tracking-live-projector-v1` consumer group, Redis availability and
the `tracking.telemetry.ingested.v1.dlt` topic. Correct poison records through the governed replay
process. For Redis outages, restore Redis and allow the uncommitted Kafka record to retry; do not
manually create live keys, alter Tenant indexes or copy telemetry between Tenants.

## Historical telemetry operations

TS04 persists accepted canonical telemetry asynchronously to TimescaleDB. It adds no operator UI or
public history endpoint. Operators monitor `tracking-telemetry-persister-group` lag, database health,
Timescale compression/retention jobs and the access-controlled DLT. A database outage leaves offsets
uncommitted; restore TimescaleDB at V87 and allow Kafka redelivery to drain safely. Do not manufacture
history from Redis, delete backlog, copy records between Tenants or expose precise location in logs.

History uses seven-day chunks, compression after seven days and 180-day raw retention. The consumer
can be disabled during recovery without deleting accepted history. Migration or policy correction
requires a separately reviewed forward migration; V87 must not be edited or removed.

At V91, accepting a retained history fact also records durable geofence, speed and route-deviation
evaluation work before Kafka acknowledgement. Operators may disable the evaluation worker while rows remain
durable, then restore it to drain due work. Monitor pending/failed counts, oldest due time and lease expiry;
do not delete dispatch rows, manufacture legacy positions, copy work between Tenants or store raw exception
text. Failed evaluation does not remove accepted history. Idle evaluation is not enabled by this mechanism.
