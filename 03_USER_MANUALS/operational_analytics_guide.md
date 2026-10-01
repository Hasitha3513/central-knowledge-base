# Operational Analytics & Forecasting User Guide

## 1. Overview
The Operational Analytics & Forecasting module provides tenant-scoped operational visibility, capacity projections, predictive fleet maintenance scoring, and actionable recommendations.

## 2. Navigation & Access
- **Menu Location:** `Operations > Operational Analytics` (Route: `/operations/analytics`).
- **Required Permissions:** `REPORT_VIEW` or `DASHBOARD_VIEW`.

## 3. Key Capabilities

### A. Actual Operational KPIs (Recorded Facts)
- **Fleet Availability Rate:** `(Available / Total) * 100%` with vehicle counts.
- **Driver Utilization Rate:** `(Active / Total) * 100%` with driver counts.
- **Trip Completion Rate:** `(Completed / Total) * 100%` within the historical window.
- **On-Time Dispatch Rate:** Punctual dispatch percentage against 90% benchmark target.
- **Active Exceptions:** Unresolved operational incidents requiring immediate triage.
- *All recorded metrics are explicitly tagged with `ACTUAL DATA`.*

### B. Predictive Operational Risk Profile
- **Aggregate Risk Index (0–100):** Weighted risk composite combining exception severity, high maintenance risk vehicles, and dispatch delays.
- **Risk Classification:** `LOW`, `MEDIUM`, `HIGH`, or `CRITICAL`.
- **Projected Weekly Demand:** Forecasted trip volume for the next 7 days.
- **High Maintenance Risk Vehicles:** Vehicles approaching service odometer/time limits.
- *All forward projections are explicitly tagged with `PREDICTIVE FORECAST`.*

### C. Time Series Trends & Demand Projections
- **Historical Operations:** Daily volume, completions, active resources, and exceptions.
- **Demand Projections:** Daily volume projections with confidence score and trend direction (`UPWARD`, `DOWNWARD`, `STABLE`).

### D. Predictive Maintenance Risk Matrix
- Lists vehicles with upcoming maintenance needs, current odometer, projected odometer at due, estimated days until service, primary risk factors, and risk levels (`LOW`, `MEDIUM`, `HIGH`, `CRITICAL`).

### E. Actionable Recommendations
- Advisory insights categorized by domain (`FLEET_MAINTENANCE`, `DRIVER_UTILIZATION`, `CAPACITY_PLANNING`) with priority level and recommended action.
- *Recommendations are strictly advisory and require operator confirmation.*

## 4. Date Range & Forecast Horizon Filtering
- **Historical Windows:** Preset buttons for `7 Days`, `30 Days`, `90 Days`, or custom RangePicker.
- **Forecast Horizons:** Selectable ahead horizons of `7 Days`, `14 Days`, or `30 Days`.

## 5. Transport Operations Command Center (DASH-03 and DASH-04)

Open the Dashboard route (`/`) with `DASHBOARD_VIEW` to see the Transport Operations Command Center. The command-center header shows the Tenant-local reporting date, timezone and last successful evaluation time. Use **Refresh** to request only the dashboard summary; the page also refreshes every 90 seconds while it is visible and online. If a refresh fails after data was loaded, the last successful values remain visible with a stale-data warning.

The KPI strip contains:

- **Vehicles:** total Vehicles plus available, allocated and maintenance counts.
- **Available Drivers:** available Drivers plus total, assigned and unavailable counts.
- **Active Trips:** the server-authoritative active count plus scheduled, pending and problem counts.
- **Deliveries Today:** Tenant-local scheduled deliveries, completed count and on-time rate when the Delivery section is available.
- **Fuel Today:** consumed and issued quantities shown separately with their source-quality labels and server-provided unit.
- **Open Exceptions:** all non-terminal operational exceptions and available severity detail.

A genuine available count of zero is shown as `0`. **Restricted** means the actor lacks the specialized permission; **Data unavailable** means the owning source could not provide the section. Neither state is converted to a false zero. Fleet, Driver and Trip summaries require only `DASHBOARD_VIEW`; each drill-down link additionally requires its destination permission. Dashboard visibility never replaces backend authorization.

Users who also hold `TRACKING_DASHBOARD_VIEW` see the **Live Operations** region below the KPI strip. It shows Tracking-owned Vehicle freshness, connectivity, observed motion, active-Trip context and permission-filtered incident counts. The map uses only complete `TRUSTED` `latestTrusted` observations. Exact coordinates and the rendered map require `TRACKING_VIEW`; without it, the accessible Vehicle list and safe non-coordinate status remain available. Selecting **View details** opens the bounded Vehicle drawer with Tracking health, source/receipt times, optional speed and accuracy, Trip context and permitted navigation actions.

The Vehicle list remains usable on narrow screens and whenever the map style cannot load. A map failure is reported as **Map unavailable** without hiding list data; missing coordinate permission is reported as **Map restricted**. No map-provider secret is stored in the frontend. Operators configure the deployment-approved style URL and attribution through the documented environment values. DASH-05 adds four visual analytics panels below Live Operations. **Fleet Availability** presents available, allocated, maintenance and out-of-service counts as a ring with a text legend. **Trip Operations** presents scheduled, active, completed and problem counts as bounded bars. **Driver Availability** presents available, assigned and unavailable counts as a segmented bar with explicit totals. **Fuel Management** keeps consumed-today and issued-today quantities independent and shows the server-provided unit and source-quality metadata for each measure. Visual percentages and bar widths are presentation aids derived from the returned counts; they do not replace the server-authoritative values.

A genuine zero remains `0`, while restricted and unavailable sections remain visibly distinct. Each panel reuses the command-center summary response and performs no additional analytics request. Fleet, Trip, Driver and Fuel drill-down links appear only when the user has the corresponding destination permission (`VEHICLE_VIEW`, `TRIP_VIEW`, `DRIVER_VIEW` or `FUEL_PERFORMANCE_VIEW`); `DASHBOARD_VIEW` remains limited to the core summary. The panels use a two-column desktop layout and stack on narrow screens. DASH-06 adds the **Delivery Performance** and **Operational Exceptions** action region. Delivery Performance reuses the existing command-center summary and shows scheduled, completed and on-time delivery counts. Its completion progress is a presentation aid only; the dashboard does not infer an on-time percentage when the server provides only counts. The Delivery drill-down appears only with `DELIVERY_VIEW`.

Operational Exceptions uses the existing bounded dashboard-alerts feed. It displays only Operations-owned `OPERATIONAL_EXCEPTION` alerts, preserves server ordering, and limits the visible list to eight items. Tracking incidents remain in the Tracking safety panel and are not duplicated. The panel is requested and shown only with `OPERATIONAL_EXCEPTION_VIEW`; the supported action target remains `/operations/exceptions`. Restricted, unavailable, stale and genuine-empty states remain distinct, and severity is always stated in text as well as color. On narrow screens the two panels stack vertically. Later workflow zones remain unavailable until their separately governed dashboard slices are implemented.
