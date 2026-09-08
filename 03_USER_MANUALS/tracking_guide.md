# Live Vehicle Tracking Guide

## Prerequisites and access

Open **Live Tracking** from the application navigation. `TRACKING_VIEW` is required for current/last-known positions and device reference summaries; `TRACKING_HISTORY_VIEW` additionally permits bounded history; `TRACKING_DEVICE_MANAGE` permits device registration, lifecycle and Vehicle association actions. Backend Tenant isolation remains authoritative.

## Vehicle positions

The Vehicle positions tab refreshes every 15 seconds while the page is visible and online. Each row labels the position as **CURRENT** or **LAST KNOWN**, and shows coordinates, source timestamp/age, accuracy, trust, freshness and connectivity. Stale data is never presented as live. Open history to inspect up to 24 hours ordered newest first; larger windows are rejected.

## Tracking devices

Authorized device managers can register a provider-neutral external device reference, optionally record a hardware serial reference, activate/disable the device, and start/end an effective-dated Vehicle association. One device and one Vehicle may each have only one active association. View-only users see masked reference values and never see provider credentials.

## Known limitations

US-48 supplies foundational live/last-known facts only. Geofences, speeding, idle detection, route deviation, journey replay, the full tracking dashboard, GPS exception workflows and customer location exposure are not available. Automatic retention purge is disabled until an external legal retention policy is configured. Final acceptance still requires a physical GPS device and real provider payload.
