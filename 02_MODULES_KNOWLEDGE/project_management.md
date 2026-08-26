# Project Management — Blueprint

Lifecycle: PROPOSED.

## Scope and Ownership

Owns project master, work breakdown structures, milestones, schedules, assignments, risks/issues, budgets at project intent level, time/cost collection, and project status. Finance owns accounting postings and authoritative actuals.

## Candidate Aggregates and Use Cases

- Project: sponsor, manager, dates, status, customer/program references.
- Work Breakdown Structure: tasks, dependencies, milestones, progress.
- Resource Plan: role/employee/equipment demand and assignments.
- Project Budget/Forecast: approved baseline and revisions.
- Risk and Issue: ownership, severity, mitigation, resolution.
- Initiate, plan, baseline, assign, update progress, forecast, suspend, complete, and close.

## Integration Points

Consumes employee availability, financial actuals/budget control, procurement commitments, inventory issues, and transportation service facts. Publishes project/status/cost-code/resource-demand/milestone events and provides project/cost-code validation.

## Legacy Reconciliation

Transportation currently owns `project` with a department reference and uses `project_id` on trips. Project ownership migration requires an ADR, stable UUID mapping, compatibility API, and logical references replacing physical cross-module constraints.

## Candidate Tables (Not Approved)

`project`, `work_item`, `dependency`, `milestone`, `resource_plan`, `resource_assignment`, `project_budget`, `project_forecast`, `risk`, `issue`, `project_status_history`.

## Open Decisions

Portfolio/program scope, scheduling engine, earned value, billing projects, baseline rules, and ownership of project budget versus Finance.
