# Human Resource Management — Blueprint

Lifecycle: PROPOSED.

## Scope and Ownership

Owns person/employee master, employment, organization assignment, positions, contracts, qualifications, leave, attendance, performance, payroll inputs, and workforce compliance. Authentication identities remain platform-owned. Transportation-specific driving assignments remain transportation-owned.

## Candidate Aggregates and Use Cases

- Employee: identity linkage, employment number, contacts, status, effective-dated assignment.
- Position and Organization Assignment: department, manager, location, cost center.
- Qualification: licence/certification type, validity, restrictions, verification.
- Leave and Availability: request, approval, calendar, conflicts.
- Time and Attendance: shifts, timesheets, approvals.
- Hire, transfer, suspend, terminate, qualify, renew, request/approve leave, record time, export payroll inputs.

## Integration Points

Publishes employee status/assignment/qualification/availability changes. Provides tenant-scoped employee and driver-eligibility queries to Transportation, Maintenance, and Projects. Consumes project assignments and operational work/time facts. Sends payroll summaries and employee cost dimensions to Finance.

## Legacy Reconciliation

Transportation currently owns `driver`, `driver_license`, `driver_exception`, `driver_violation`, `driver_medical_record`, and `driver_drug_test`. An ADR must decide which records move to HRM, which remain transport safety records, and how IDs and historical API behavior are preserved.

## Candidate Tables (Not Approved)

`employee`, `employment`, `position`, `organization_assignment`, `qualification`, `leave_request`, `availability_period`, `shift`, `timesheet`, `timesheet_entry`. Sensitive medical and disciplinary data requires stricter authorization, encryption, audit, and retention controls.

## Open Decisions

Employment jurisdictions, payroll scope, privacy classification, employee/person separation, contingent workers, union rules, and transportation driver ownership.
