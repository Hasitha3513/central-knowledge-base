# Vehicle Maintenance — Blueprint

Lifecycle: PROPOSED.

## Scope and Ownership

Owns maintenance plans, service intervals, work requests/orders, inspections, defects, breakdowns, labor/parts execution, vendors, downtime, warranties, and return-to-service decisions. Transportation remains authoritative for vehicle master and operational readings unless an ADR changes ownership.

## Candidate Aggregates and Use Cases

- Maintenance Plan: calendar/meter triggers, task templates, applicability.
- Work Order: vehicle, cause, priority, status, tasks, labor, parts, costs.
- Inspection/Defect: findings, severity, safety hold, resolution.
- Breakdown: incident, recovery, diagnosis, downtime.
- Schedule, trigger, approve, start, pause, complete, cancel, record labor/parts, place/release hold, and forecast maintenance.

## Integration Points

Consumes vehicle master/status, meter readings, trip utilization, driver defect reports, parts availability, purchase-order status, and financial cost-posting outcomes. Publishes maintenance due, vehicle hold/release, work-order lifecycle, parts demand/consumption, downtime, and cost facts. Provides vehicle dispatch eligibility through a focused API/port.

## Legacy Reconciliation

Transportation currently owns `maintenance_schedule` and uses it in vehicle assignment blocking. Extraction requires characterization tests, an ADR for transaction/availability semantics, event/API compatibility, and no interruption to offline vehicle-reading integration.

## Candidate Tables (Not Approved)

`maintenance_plan`, `maintenance_interval`, `work_request`, `work_order`, `work_order_task`, `labor_entry`, `part_requirement`, `part_consumption`, `inspection`, `defect`, `breakdown`, `vehicle_hold`, `warranty_claim`, `maintenance_status_history`.

## Invariants

Safety-critical holds override dispatch; work completion requires required inspections/tasks; meter-trigger processing is idempotent; parts are logical Inventory references; financial postings are asynchronous and traceable.

## Open Decisions

Maintenance-schedule migration, workshop/vendor model, warranty scope, labor source, cost capitalization, telematics ingestion, and emergency override policy.
