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

## 5. Transport Operations Command Center (DASH-03)

Open the Dashboard route (`/`) with `DASHBOARD_VIEW` to see the Transport Operations Command Center. The command-center header shows the Tenant-local reporting date, timezone and last successful evaluation time. Use **Refresh** to request only the dashboard summary; the page also refreshes every 90 seconds while it is visible and online. If a refresh fails after data was loaded, the last successful values remain visible with a stale-data warning.

The KPI strip contains:

- **Vehicles:** total Vehicles plus available, allocated and maintenance counts.
- **Available Drivers:** available Drivers plus total, assigned and unavailable counts.
- **Active Trips:** the server-authoritative active count plus scheduled, pending and problem counts.
- **Deliveries Today:** Tenant-local scheduled deliveries, completed count and on-time rate when the Delivery section is available.
- **Fuel Today:** consumed and issued quantities shown separately with their source-quality labels and server-provided unit.
- **Open Exceptions:** all non-terminal operational exceptions and available severity detail.

A genuine available count of zero is shown as `0`. **Restricted** means the actor lacks the specialized permission; **Data unavailable** means the owning source could not provide the section. Neither state is converted to a false zero. Fleet, Driver and Trip summaries require only `DASHBOARD_VIEW`; each drill-down link additionally requires its destination permission. Dashboard visibility never replaces backend authorization.

The current slice provides the responsive command-center header and KPI strip. Map, chart, alert-detail and workflow zones remain unavailable until their later governed dashboard slices are implemented.
