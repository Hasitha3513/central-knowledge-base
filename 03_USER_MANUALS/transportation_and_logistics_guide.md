# Transportation and Logistics End-User Operational Manual

**Audience:** Fleet Managers, Dispatchers, Drivers, Fuel Officers, and authorized operations staff  
**Scope:** Phase 1 and Phase 2A transportation operations, plus the implemented Phase 3 core fuel workflows  
**System area:** Transportation and Logistics

## 1. Purpose and Scope

This manual explains how to complete the day-to-day transportation and logistics workflows available in the current system. It covers vehicle records and compliance documents, trip-based vehicle allocation, trip execution, route planning, driver records, and core fuel operations.

The system is permission-controlled and tenant-scoped. A user sees only the menu items, records, and actions allowed by their assigned role and organization membership. If an action described here is not visible, contact an administrator instead of using another user's account.

### Current-scope clarifications

- **Vehicle reservation:** In the current UI, reserving a vehicle means assigning an eligible vehicle to a trip. There is no separate provisional-hold or allocation-calendar workflow.
- **Compliance document upload:** The current workflow records document metadata and a file reference or URL. It does not transfer a binary file into an internal document store.
- **Route optimization:** Optimization produces a preview. The proposed stop order is not operational until an authorized user applies it.
- **Bunker dip reading:** A dip is a physical observation. Recording it does not change the authoritative book stock. Use an authorized stock adjustment to reconcile a verified difference.
- **Phase 3 fuel scope:** Fuel issues, purchases, prices, bunker inventory, vehicle readings, and mileage summaries are available. Deferred advanced fuel capabilities are outside this manual.

## 2. Getting Started

### Sign in and open a module

Step 1. Open the application and sign in with your assigned username and password.

Step 2. Confirm that the application workspace opens and that your organization context is correct.

Step 3. Use the left navigation menu to open **Fleet**, **Drivers**, **Routes**, **Trips**, or **Fuel**.

Step 4. If a module or action is absent, ask an administrator to verify your membership and permissions. Do not repeatedly retry an unauthorized action.

### Common screen behavior

- Use list filters and search fields before creating a record to avoid duplicates.
- Select a row or its action menu to open details, edit, or lifecycle actions.
- Required fields must be completed before **Save**, **Continue**, or **Confirm** becomes available.
- A success message confirms that the server accepted the operation. Keep the displayed correlation ID when reporting an error.
- Refresh the list after another operator changes the same record.

## 3. Vehicle Master and Compliance Documents

**Navigation:** **Fleet > Vehicles** (`/fleet/vehicles`)  
**Related setup:** **Fleet > Vehicle Categories** and **Fleet > Vehicle Types**

### 3.1 Register a vehicle

Before starting, confirm that the correct vehicle category and type already exist.

Step 1. Open **Fleet > Vehicles**.

Step 2. Select **Register Vehicle** or **Add Vehicle**.

Step 3. Enter the vehicle's identifying details, including registration number and the available manufacturer/model information.

Step 4. Select the appropriate vehicle type, category, ownership classification, and capacity information.

Step 5. Enter the current odometer value when it is known and verified.

Step 6. Set the operational status. Use **Available** only when the vehicle is serviceable and its mandatory compliance records are valid.

Step 7. Review the information and select **Save**.

Step 8. Reopen the vehicle details and confirm that the registration number, classification, capacity, ownership, and status are correct.

### 3.2 Record an RC, Insurance, or Emission document

Step 1. Open **Fleet > Vehicles** and select the vehicle.

Step 2. Open the vehicle's **Documents** or compliance section.

Step 3. Select **Add Document**.

Step 4. Choose the document type, such as **RC**, **Insurance**, or **Emission**.

Step 5. Enter the document number, issue date, and expiry date.

Step 6. Enter the approved file reference or URL. Confirm that authorized users can access the referenced document.

Step 7. Enable **Mandatory for dispatch** when the document is required before the vehicle can operate.

Step 8. Set the document as active and save it.

Step 9. Review the vehicle details and confirm that the document type, number, expiry date, active state, and mandatory-for-dispatch flag are correct.

### 3.3 Handle expiry alerts

Step 1. Review the Fleet dashboard's expiring-document alerts regularly.

Step 2. Open the affected vehicle and verify the document and expiry date.

Step 3. Arrange renewal outside the system using the organization's compliance process.

Step 4. Add or update the renewed document record with the new number, dates, and file reference.

Step 5. Deactivate the superseded record when required by local procedure, while retaining its history.

Step 6. Confirm that the vehicle is eligible for dispatch. Do not mark it **Available** if a mandatory document remains missing or expired.

### Troubleshooting / Common Pitfalls

- **Duplicate registration:** Search by registration number before creating a vehicle. Correct the existing record rather than creating a second vehicle.
- **Missing document at dispatch:** Verify that all required document records are active, unexpired, and marked correctly. A file reference alone is not enough if mandatory metadata is missing.
- **Document appears “uploaded” but cannot be opened:** Check the stored file reference or URL and the user's access to the external location.
- **Vehicle not available:** Check its status, active maintenance, mandatory documents, and any existing trip assignment.
- **Incorrect odometer:** Do not enter a lower value to represent a replacement meter. Use the governed meter-reset workflow so lifetime mileage remains continuous.

## 4. Vehicle Allocation

**Navigation:** **Trips > select a trip > Assign Vehicle** (`/trips/:id`)

### 4.1 Check vehicle availability

Step 1. Open **Trips** and select the trip requiring a vehicle.

Step 2. Confirm that the trip's dates, route, cargo/capacity needs, and lifecycle state are accurate.

Step 3. Select **Assign Vehicle**.

Step 4. Review the candidate vehicles and their availability or eligibility result.

Step 5. Select a vehicle only when the system marks it available and its operational capacity is suitable.

Step 6. If a vehicle is unavailable, review the reason shown by the system before choosing another candidate.

### 4.2 Reserve a vehicle for a trip

Step 1. In the assignment drawer, select an eligible vehicle.

Step 2. Select **Continue**.

Step 3. Review the trip and vehicle in the confirmation dialog.

Step 4. Select **Confirm Assignment**.

Step 5. Wait for the **Vehicle assigned** confirmation and verify that the vehicle now appears on the trip.

The confirmed trip assignment is the current operational reservation. To change it, use the trip's authorized unassign/reassign action; do not create a duplicate trip.

### 4.3 Resolve allocation conflicts or overbooking

Step 1. Read the reason in the assignment error or availability result.

Step 2. Recheck the trip schedule and the vehicle's existing assignments, status, maintenance, documents, and capacity.

Step 3. Choose one of the permitted remedies:

- Select another eligible vehicle.
- Correct an incorrect trip schedule before retrying.
- Complete, cancel, or correct the conflicting trip through its proper lifecycle action.
- Resolve expired documents or maintenance restrictions before returning the vehicle to service.

Step 4. Refresh the trip and retry the assignment once the underlying condition is resolved.

Step 5. If another operator changed the record at the same time, reload the latest trip state before retrying.

### Troubleshooting / Common Pitfalls

- **Continue button disabled:** Select a vehicle that the system reports as available.
- **Vehicle looked available but assignment failed:** Another assignment may have been confirmed concurrently. Refresh and select from the latest candidates.
- **Overbooking:** Do not bypass the eligibility result. Resolve the schedule conflict or use a different vehicle.
- **Wrong-capacity vehicle:** Availability does not replace operational judgment. Confirm payload, volume, and equipment needs before assignment.
- **No eligible vehicles:** Escalate to the Fleet Manager with the trip dates, required capacity, and displayed rejection reasons.

## 5. Trip Lifecycle

**Navigation:** **Trips** (`/trips`), **New Trip** (`/trips/new`), and **Trip Details** (`/trips/:id`)

Only lifecycle actions valid for the current status and the user's permissions are displayed. Never treat a hidden action as a reason to skip a required approval.

### 5.1 Create the trip

Step 1. Open **Trips** and select **New Trip**.

Step 2. Enter the planned movement details, dates/times, origin and destination information, and other required references.

Step 3. Save the trip as a draft.

Step 4. Open the trip details and verify the plan before submitting it.

### 5.2 Assign the route, vehicle, and driver

Step 1. From Trip Details, select the route assignment action and choose the approved route.

Step 2. Select **Assign Vehicle**, choose an eligible candidate, continue, and confirm.

Step 3. Select **Assign Driver**.

Step 4. Choose the required licence class and review the driver's availability result.

Step 5. Select an eligible driver, continue, and confirm the assignment.

Step 6. Confirm that the route, vehicle, and driver all appear on the trip.

### 5.3 Submit and authorize

Step 1. Review the route, schedule, assignments, and mandatory vehicle compliance records.

Step 2. Select **Submit** to send the draft for authorization.

Step 3. An authorized approver reviews the trip and selects **Approve** or **Reject**.

Step 4. If rejected, read the reason, correct the draft through the permitted workflow, and resubmit.

Step 5. If approved, confirm that the trip is ready for dispatch and that no assignment or compliance block remains.

### 5.4 Dispatch and start

Step 1. Immediately before departure, recheck the assigned driver, vehicle, route, documents, and operational readiness.

Step 2. Select **Dispatch** and confirm the command.

Step 3. When physical movement begins, use **Start** if it is presented as a separate action.

Step 4. Confirm that the trip status reflects the real-world operation. Do not dispatch or start a trip merely to clear a queue.

### 5.5 Maintain trip logs

Step 1. Open the trip's **Trip Logs** or operational section.

Step 2. Record checkpoints and operational events at the time they occur.

Step 3. Record delays with accurate timestamps and reasons.

Step 4. Record incidents promptly and follow the organization's separate safety/escalation procedure.

Step 5. Keep route and operational notes factual. Do not overwrite earlier history to correct a later event.

### 5.6 Complete and close

Step 1. Confirm that the vehicle and cargo have reached the intended destination and required operational logs are present.

Step 2. Record final operational information, including readings where required.

Step 3. Select **Complete** and verify that completion is accepted.

Step 4. An authorized user performs any final review and selects **Close** when that action is required.

Step 5. Use **Cancel** only for a trip that will not proceed, and record the genuine reason.

### Troubleshooting / Common Pitfalls

- **Cannot submit:** Complete the required trip fields and reload the latest record.
- **Cannot assign a driver:** Check required licence class, licence validity, medical/assignment blocks, exceptions/leave, and conflicting trips.
- **Cannot dispatch:** Check approval state, vehicle and driver assignments, route, mandatory documents, vehicle condition, and other displayed readiness failures.
- **Lifecycle action missing:** The current status may not permit it, or your role may lack permission. Do not try to change the status through editing.
- **Stale update or version conflict:** Another operator changed the trip. Refresh, review the new state, and repeat only the still-valid action.
- **Incorrect event time:** Report it through the approved correction process; do not create misleading duplicate events.

## 6. Basic and Advanced Routes

**Navigation:** **Routes** (`/routes`)

### 6.1 Create a basic route

Step 1. Open **Routes** and select **Add Route** or **Create Route**.

Step 2. Enter the route code/name and the origin and destination information.

Step 3. Enter planned distance and duration when available.

Step 4. Save the route and reopen its details to verify the data.

Step 5. Activate or use the route only after it has passed the organization's operational review.

### 6.2 Create a multi-stop route

Step 1. Open the route and its stops or revision section.

Step 2. Add each approved stop location.

Step 3. Arrange the stops in the intended operational sequence.

Step 4. Review distance, duration, timing, and practical access constraints.

Step 5. Save the revision and confirm the stop order.

Step 6. If optimization is available, select **Optimize Route** to calculate a preview.

Step 7. Compare the proposed stop sequence, distance, and duration with the current plan.

Step 8. Select **Apply Optimization** only when the proposal is operationally acceptable. Otherwise close the preview without applying it.

### 6.3 Report and resolve a route disruption

Step 1. Open the affected route and locate **Route Disruptions**.

Step 2. Select **Report Disruption**.

Step 3. Choose the disruption type, such as **Road Closure**, **Accident**, **Weather**, or **Restriction**.

Step 4. Select the severity—**Low**, **Medium**, **High**, or **Critical**—and enter a factual description.

Step 5. Save the disruption and notify affected dispatch operations according to local procedure.

Step 6. Review impacted trips and assign or communicate an approved alternative route when necessary.

Step 7. When normal conditions are verified, select **Resolve Disruption** and confirm.

### Troubleshooting / Common Pitfalls

- **Stops appear in the wrong order:** Recheck the saved revision and sequence before assigning the route to a trip.
- **Optimization preview is unsuitable:** Do not apply it. Optimization is decision support and does not replace access, safety, vehicle, or regulatory judgment.
- **Disruption still shown after clearance:** An authorized user must explicitly resolve it after verifying conditions.
- **Active trip uses an outdated plan:** Coordinate with Dispatch before changing route information; preserve operational history and use the approved reassignment/revision process.

## 7. Driver Management

**Navigation:** **Drivers** (`/drivers`)

### 7.1 Create or update a driver profile

Step 1. Open **Drivers**.

Step 2. Search by employee/driver identifier and name to avoid duplicates.

Step 3. Select **Add Driver** for a new record, or open an existing driver and select **Edit**.

Step 4. Enter the required identity, contact, employment, and operational information.

Step 5. Set the correct active/status value and save.

Step 6. Reopen the profile and verify the saved information before assigning the driver.

### 7.2 Manage driver licences

Step 1. Open the driver profile and its **Licences** section.

Step 2. Select **Add Licence**.

Step 3. Enter the licence number, class, issue date, and expiry date.

Step 4. Record any available supporting reference and save.

Step 5. Review expiring licences regularly and update renewal information before assignment.

Step 6. Confirm that the required licence class is valid before assigning the driver to a vehicle or trip.

### 7.3 Manage medical records

Step 1. Open the driver profile and select **Medical Records**.

Step 2. Select **Add Medical Record**.

Step 3. Enter the examination date, outcome/status, expiry or next-review date, and appropriate notes.

Step 4. Save and verify the record.

Step 5. If the result creates an assignment restriction, follow the organization's safety and return-to-duty procedure. Do not manually treat a blocked driver as eligible.

Medical information is sensitive. Access it only for an authorized operational purpose and avoid placing unnecessary clinical details in general notes.

### 7.4 Review driver performance

Step 1. Open the driver profile and select **Performance**.

Step 2. Review the overall rating, safety score, trip completion rate, assigned/completed/cancelled trip counts, violations, penalty points, fines, and unpaid fines.

Step 3. Check the evaluation timestamp so that you understand how current the scorecard is.

Step 4. Investigate source records before acting on an unexpected value.

Step 5. Use the scorecard as operational decision support; follow HR, safety, and disciplinary policy for formal action.

### Troubleshooting / Common Pitfalls

- **Driver not eligible for assignment:** Check licence class and expiry, medical/return-to-duty status, drug-test blocks, leave/exceptions, active status, and trip conflicts.
- **Duplicate driver:** Search existing employee and licence identifiers before creating a profile.
- **Performance value looks incorrect:** Check the evaluation time and underlying trips or violations, then raise a data-correction request.
- **Sensitive notes visible to the wrong audience:** Stop adding data and report the access concern to an administrator or privacy owner.

## 8. Fuel Operations — Implemented Phase 3 Core

**Navigation:**

- **Fuel > Fuel Issues** (`/fuel/issues`)
- **Fuel > Fuel Purchases** (`/fuel/purchases`)
- **Fuel > Bunker Tanks** (`/fuel/bunker-tanks`)
- **Fuel > Fuel Prices** (`/fuel/prices`)
- **Fleet > Vehicles > Vehicle Details > Readings** for odometer, engine-hours, and mileage

### 8.1 Log a vehicle fuel refill through Fuel Issues

Step 1. Open **Fuel > Fuel Issues** and select **New Fuel Issue**.

Step 2. Select the correct vehicle, fuel station/source, and fuel type.

Step 3. Enter the issued quantity and other required transaction details.

Step 4. Enter the verified odometer and engine-hours readings when applicable.

Step 5. Review the unit, quantity, reading values, vehicle, and source before saving the draft.

Step 6. Select **Submit** when the request is complete.

Step 7. An authorized user selects **Authorize** after checking entitlement and availability.

Step 8. At physical dispensing, select **Issue** and verify that the transaction reaches its issued state.

Step 9. Use **Cancel** only when dispensing will not occur and the current lifecycle permits cancellation.

### 8.2 Record and review odometer or mileage readings

Step 1. Open **Fleet > Vehicles**, select the vehicle, and open **Readings**.

Step 2. Review the latest odometer, engine-hours, meter epoch, and reading time.

Step 3. Select the action to add a reading.

Step 4. Enter the observed value, correct unit, timestamp, and required source/reference information.

Step 5. Save and confirm that the latest-reading panel is updated.

Step 6. Review **Mileage** for total distance, hours used, coverage status, reset count, and abnormal-reading warnings.

Step 7. Use the correction action for a recognized data-entry error. Use the meter-reset workflow for a physical meter replacement or rollover.

### 8.3 Record a fuel purchase

Step 1. Open **Fuel > Fuel Purchases** and select **New Purchase**.

Step 2. Enter supplier, fuel, quantity, price/cost, receiving location, and required reference details.

Step 3. Save and review the draft.

Step 4. Follow the displayed lifecycle actions: **Submit**, **Approve**, **Receive**, and **Reconcile**.

Step 5. Confirm physical receipt and supporting documents before selecting **Receive**.

Step 6. Reconcile the transaction only after amounts and received stock have been checked.

### 8.4 Manage bunker transactions

#### Create a bunker tank and establish stock

Step 1. Open **Fuel > Bunker Tanks**.

Step 2. Select **Add Bunker Tank**, choose the internal station, and enter the tank details and capacity.

Step 3. Save the tank.

Step 4. Use **Opening Balance** only when initializing an eligible new tank and enter the verified starting quantity.

#### Record a physical dip

Step 1. Open the bunker tank details.

Step 2. Select **Record Dip Reading**.

Step 3. Enter the physically measured quantity, measurement time, and notes.

Step 4. Save and compare the physical observation with the book balance.

Step 5. If the difference is genuine, investigate it before creating an authorized stock adjustment. The dip itself does not change stock.

#### Adjust or transfer stock

Step 1. Open the affected tank and review its balance and movement history.

Step 2. For a correction, select the adjustment action and enter the quantity, direction/reason, and evidence.

Step 3. For a transfer, select the source and destination tanks and enter the transfer quantity.

Step 4. Verify that sufficient stock exists and that the destination is correct.

Step 5. Confirm the transaction and review both tanks' movement histories and balances.

### Troubleshooting / Common Pitfalls

- **Fuel issue cannot be authorized or issued:** Check its current lifecycle state, vehicle/source selection, quantity, user permission, and available inventory.
- **Odometer rejected:** Confirm the unit and latest reading. Use correction or meter reset rather than entering an unexplained lower reading.
- **Mileage coverage is incomplete:** Check for missing time periods, missing readings, or meter-reset history before using the summary for reporting.
- **Dip differs from book stock:** Re-measure and investigate receipts, issues, transfers, and timing. Never use repeated dip entries to force the balance.
- **Adjustment exceeds available stock:** Correct the quantity or investigate the balance; do not split the transaction to bypass validation.
- **Wrong tank in a transfer:** Stop before confirmation. After confirmation, use the approved correction/escalation process rather than inventing an offsetting transaction without authorization.

## 9. Delivery Orders & Proof of Delivery (POD)

**Navigation:** **Delivery > Delivery Orders** (`/deliveries`)

### 9.1 Create and Validate a Delivery Order
Step 1. Navigate to **Delivery > Delivery Orders** and click **New delivery order**.
Step 2. Select Customer, Origin Location, Destination Location, Priority (Low, Normal, High, Urgent), Service Type (Standard, Express, Same Day, Scheduled), Delivery Window (Start/End), and optional Special Instructions.
Step 3. Click **Save delivery order**. An immutable server-generated identifier formatted `DEL-YYYY-NNNNNN` is allocated.
Step 4. Review the draft order and click **Validate readiness**. If all customer and location references are active within the Tenant, the order transitions from `DRAFT` to `READY FOR ASSIGNMENT`.

### 9.2 Capture Proof of Delivery (POD) (Online & Offline)
Step 1. Open a Delivery Order in `READY FOR ASSIGNMENT` status.
Step 2. In the **Proof of Delivery** section, enter the recipient Signer Name (mandatory for signatures) and optional Signer Relationship.
Step 3. **Customer Consent (POD-CONSENT-V1):** Explicitly check the consent confirmation checkbox before capturing electronic signature or photo evidence.
Step 4. (Optional) Click **Capture location** to capture device latitude and longitude.
Step 5. Attach primary delivery evidence:
- **Draw Signature:** Open the signature canvas to draw signature directly on touch screen or with mouse. Use **Clear / Retake** if corrections are needed.
- **Upload Signature:** Upload a PNG/JPEG file (<= 2MB).
- **Delivery Photo:** Capture via camera or upload up to 3 PNG/JPEG photos (<= 10MB each).
- **Barcode Scan:** Scan or enter the exact Delivery Order number barcode (`DEL-YYYY-NNNNNN`).
Step 6. Finalization & Sync Modes:
- **Online Mode:** Click **Finalize POD Online** to atomically submit directly to the server.
- **Offline / Staged Mode:** Click **Save & Queue Offline** to persist the complete POD package into the local browser IndexedDB outbox (`DELIVERY_POD_OFFLINE_SYNC`). The system automatically synchronizes the queued POD when connectivity is restored.
Step 7. Upon successful server processing:
- Proof of Delivery transitions to `FINALIZED` and becomes permanently immutable.
- Delivery Order atomically transitions from `READY FOR ASSIGNMENT` to `DELIVERED`.

### 9.3 Record a Failed Delivery Attempt (US-59)
Step 1. Open a Delivery Order in `READY FOR ASSIGNMENT` status when delivery could not be completed in the field.
Step 2. In the **Failed Delivery Attempt** section, select a standardized **Failure Reason**:
- `Customer Unavailable`, `Wrong Address`, `Access Restricted`, or `Document/Payment Issue` (defaults to `Redelivery Eligible`).
- `Customer Refused` (defaults to `Return to Base Required`; requires explanation notes $\ge 5$ chars).
- `Damaged Cargo` (defaults to `Escalated`; requires explanation notes $\ge 5$ chars).
- `Other` (requires detailed explanation notes $\ge 10$ chars).
Step 3. (Optional) Record customer contact attempts made in the field (Phone, SMS, WhatsApp, Email, or In Person) along with outcome (e.g. `No Answer`, `Line Busy`, `Spoke to Customer`).
Step 4. Click **Record Failed Attempt**. The order transitions to `FAILED ATTEMPT`, `RETURN TO BASE`, or `ESCALATED` according to the failure disposition.
Step 5. For orders requiring operational management resolution, use **Escalate** or **Return to Base** with supervisor confirmation.

### 9.4 Schedule Re-Delivery (US-60)
Step 1. Open a Delivery Order in `FAILED ATTEMPT` status whose latest attempt disposition is `Redelivery Eligible`.
Step 2. In the **Re-Delivery Scheduling** section, click **Schedule Re-Delivery**.
Step 3. Choose a scheduling approach:
- **Automatic / Depot Suggestions:** Click **Get Available Slot Suggestions** to view capacity-verified morning (09:00–13:00) and afternoon (14:00–18:00) next-day slots, then click a slot to select it.
- **Customer Preference Window:** Enter customer requested start/end times and advisory notes, then click **Get Available Slot Suggestions** to check whether the preferred window fits within operational depot hours (08:00–20:00) and concurrent capacity ($\le 50$).
- **Agent-Assisted Custom Window:** Select a valid window within the 30-day scheduling horizon directly using the date-time range picker.
Step 4. Click **Confirm & Schedule**.
Step 5. The delivery order atomically transitions from `FAILED ATTEMPT` back to `READY FOR ASSIGNMENT`, updating the active delivery window and recording an immutable `CONFIRMED` audit schedule entry.
Step 6. **Rescheduling:** If the customer requests a different time window while the order is in `READY FOR ASSIGNMENT`, click **Reschedule**, select a new capacity-verified window with a reason for change, and confirm. The previous schedule is marked `SUPERSEDED` and the new window becomes active.

## 10. Daily Operational Checklists

### Fleet Manager

- Review unavailable vehicles, active maintenance, and expiring mandatory documents.
- Confirm that newly registered vehicles have correct classifications and compliance records.
- Review abnormal odometer/mileage readings and resolve them through governed corrections.

### Dispatcher

- Review today's trips, lifecycle states, and route disruptions.
- Confirm route, eligible vehicle, eligible driver, and approval before dispatch.
- Monitor delays/incidents and ensure trips are completed and closed accurately.

### Driver / Field Delivery Agent

- Confirm assigned trip, vehicle, route, delivery orders, and departure instructions.
- Capture valid Proof of Delivery (signature, photo, or barcode) upon completion of each delivery stop.
- Report document, vehicle, route, delay, incident, and reading issues promptly.
- Provide accurate operational readings and completion information; do not share credentials.

### Fuel Officer

- Check fuel issue authorization before dispensing.
- Record quantities and meter readings from verified source information.
- Review bunker balances, physical dips, purchases, transfers, and reconciliation exceptions.

## 11. Support and Escalation

When an operation fails:

Step 1. Read the full message and correct any highlighted field.

Step 2. Refresh the record to rule out a concurrent update.

Step 3. Confirm the current status, organization context, and your permissions.

Step 4. Do not create a duplicate record or bypass a readiness, compliance, or inventory control.

Step 5. Escalate with the module, record identifier, time, intended action, displayed message, and correlation ID. Do not include passwords or unnecessary medical/personal information.

## 12. Scope Boundaries

This guide describes the currently implemented operational workflows. It does not authorize users to bypass server validation, tenant isolation, approval controls, compliance blocks, stock controls, or lifecycle rules. Capabilities identified in the roadmap as deferred or post-MVP are not operational merely because a related screen or data field exists.
