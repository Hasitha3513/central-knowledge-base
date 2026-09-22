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
