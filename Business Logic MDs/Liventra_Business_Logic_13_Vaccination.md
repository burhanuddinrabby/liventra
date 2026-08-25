# Liventra — Business Logic Documentation
## Model 13: Vaccination

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** Health & Vaccination

---

# 1. Source / Finalized Model

The finalized `Vaccination` model is:

```text
Vaccination Model:
_id
vaccinationNo
farmId
animalId
vaccineId
vaccinationDate
nextDueDate
administeredBy
batchNumber
status: Scheduled | Completed | Missed | Cancelled
remarks
createdBy
updatedAt
```

This exact structure is present in the current finalized DB model. fileciteturn13file0

The important relationship is:

```text
Vaccine
   ↓
vaccineId
   ↓
Vaccination
   ↓
Animal
```

`Vaccine` is the master definition; `Vaccination` is the animal-specific vaccination event/history.

---

# 2. Purpose

`Vaccination` records the **actual vaccination schedule/event for a particular animal**.

It answers:

> "Which vaccine is scheduled/administered to which animal, at which farm, on what date, with what batch, by whom, and what is its current status?"

Example:

```text
Animal:
AN-000125

Vaccine:
FMD Vaccine

Vaccination Date:
2026-08-17

Next Due:
2026-09-16

Administered By:
Farm Member A

Batch:
FMD-BATCH-2026-08

Status:
Completed
```

---

# 3. Vaccine vs Vaccination

This distinction is fundamental.

## Vaccine

```text
What is the vaccine?
```

Contains:

```text
Name
Manufacturer
Disease
Dosage
Route
Minimum age
Booster rule
Repeat interval
Storage requirement
```

## Vaccination

```text
What happened to this animal?
```

Contains:

```text
Animal
Farm
Vaccine
Vaccination date
Next due date
Administrator
Batch
Status
```

Therefore:

```text
Vaccine
   ↓
MASTER DATA

Vaccination
   ↓
TRANSACTION / EVENT HISTORY
```

---

# 4. Overall Health Architecture

The vaccination flow is:

```text
Disease
   ↓
recommendedVaccineIds[]
   ↓
Vaccine
   ↓
Vaccination
   ↓
Animal
```

Operationally:

```text
Vaccination
      ↓
Completed?
      ↓
Inventory consumption
      ↓
Inventory_Transaction
      ↓
Notification / Reminder
      ↓
AuditLog
```

This connects the health, inventory, notification and audit modules.

---

# 5. Field-by-Field Business Logic

# 5.1 `_id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique database identifier for the vaccination event.

Example:

```text
ObjectId(...)
```

This is different from:

```text
vaccinationNo
```

because `_id` is the database identifier while `vaccinationNo` is the human-readable business identifier.

---

# 5.2 `vaccinationNo`

### Type

String

### Required

Yes

### Example

```text
VACC-000001
```

### Purpose

Human-readable unique identifier for the vaccination record.

### Business Uses

- Search
- Reports
- UI
- Audit
- Reference
- Support/debugging

---

# 6. Vaccination Number Generation

Recommended format:

```text
VACC-{SEQUENCE}
```

Examples:

```text
VACC-000001
VACC-000002
VACC-000003
```

The backend should generate/validate uniqueness.

Do not rely on frontend-generated numbers.

---

# 7. `farmId`

### Type

ObjectId / Farm reference

### Required

Yes

### Purpose

Identifies the farm where the vaccination is scheduled/administered.

### Why Required?

The same animal identifier or vaccine can exist across different farms, and Liventra is designed around farm-level data isolation.

---

# 8. Farm Scope Validation

Before creating a Vaccination:

```text
farmId
   ↓
Animal.farmId
```

must match.

Example:

```text
Vaccination.farmId = Farm A
Animal.farmId = Farm B
```

This must be rejected.

The backend must enforce this even if the frontend already filters animals by farm.

---

# 9. `animalId`

### Type

ObjectId / Animal reference

### Required

Yes

### Purpose

Identifies the animal receiving the vaccine.

Example:

```text
animalId → AN-000125
```

One animal can have many vaccination records:

```text
Animal A
   |
   +--- Vaccination 1 → FMD
   +--- Vaccination 2 → HS
   +--- Vaccination 3 → FMD Booster
   +--- Vaccination 4 → Other vaccine
```

This creates the animal's vaccination history.

---

# 10. `vaccineId`

### Type

ObjectId / Vaccine reference

### Required

Yes

### Purpose

References the `_id` of the Vaccine master record.

Example:

```text
vaccineId → VAC-FMD-001
```

The finalized Vaccine model contains the vaccine definition, while Vaccination references it through `vaccineId`. fileciteturn13file0

---

# 11. Vaccine Reference Integrity

Before creating a vaccination:

```text
vaccineId must exist
```

and normally:

```text
Vaccine.isActive = true
```

for a new routine vaccination.

An inactive vaccine may remain referenced by historical records.

---

# 12. `vaccinationDate`

### Type

Date

### Required

Conditional on status

### Purpose

Stores the date associated with the vaccination event.

The interpretation depends on status.

### Completed

```text
Actual administration date
```

### Scheduled

```text
Planned vaccination date
```

### Missed

```text
Scheduled/planned date that was not completed
```

### Cancelled

```text
Planned date before cancellation
```

This distinction should be preserved in application logic.

---

# 13. Important Date Rule

For:

```text
status = Completed
```

the `vaccinationDate` should represent when the animal actually received the vaccine.

For:

```text
status = Scheduled
```

it represents the planned date.

This is important because:

```text
Scheduled date
```

and:

```text
Actual administration date
```

are not always the same.

---

# 14. `nextDueDate`

### Type

Date

### Required

Conditional

### Purpose

Stores the next date when another vaccination/booster/repeat dose is due for this animal.

This field belongs to `Vaccination`, not `Vaccine`.

Why?

Because:

```text
Vaccine
=
general scheduling rule

Vaccination
=
animal-specific due date
```

---

# 15. How nextDueDate Is Determined

The Vaccine master may contain:

```text
boosterRequired
boosterAfterDays
repeatIntervalDays
```

The Vaccination service uses these rules to calculate:

```text
nextDueDate
```

Example:

```text
Vaccination Date:
2026-08-17

boosterAfterDays:
30

Next Due:
2026-09-16
```

The exact date depends on the applicable configured protocol.

---

# 16. Booster vs Repeat Schedule

Do not blindly combine:

```text
boosterAfterDays
+
repeatIntervalDays
```

The application must determine which rule applies to the current vaccination stage.

Conceptually:

```text
Initial dose
     ↓
Booster rule
     ↓
Booster
     ↓
Repeat interval
     ↓
Future repeat
```

The exact protocol should be represented by the Vaccine configuration and application scheduling logic.

---

# 17. When nextDueDate Can Be Null

Possible cases:

```text
One-time vaccine
No further dose configured
Cancelled vaccination
Historical record where future schedule is not applicable
```

Therefore, `nextDueDate` should be conditionally required rather than blindly mandatory in every possible status.

---

# 18. `administeredBy`

### Type

ObjectId / User or FarmMember reference

### Required

Required for completed vaccination

### Purpose

Identifies the person who actually administered the vaccine.

Example:

```text
administeredBy → Farm Member A
```

---

# 19. AdministeredBy vs CreatedBy

These may be different.

Example:

```text
Worker A
→ administered vaccine

Manager B
→ entered the record into Liventra
```

Then:

```text
administeredBy = Worker A
createdBy = Manager B
```

This is useful for accountability.

---

# 20. Scheduled Vaccination and administeredBy

If:

```text
status = Scheduled
```

there may not yet be an administrator.

Therefore:

```text
administeredBy = null
```

is reasonable until the vaccination is actually completed.

When completed:

```text
administeredBy
```

should be populated.

---

# 21. `batchNumber`

### Type

String

### Required

Recommended for completed vaccination

### Example

```text
FMD-BATCH-2026-08
```

### Purpose

Identifies the specific vaccine batch used.

This is important for traceability.

---

# 22. Why Batch Number Matters

If a vaccine batch later needs investigation, Liventra can identify:

```text
Which animals received that batch?
```

Query conceptually:

```text
Vaccination
where batchNumber = "FMD-BATCH-2026-08"
```

This is valuable for:

```text
Traceability
Recall investigation
Quality monitoring
Inventory tracking
Farm records
```

---

# 23. Batch Number and Inventory

Inventory stores the farm's stock.

Vaccination stores:

```text
Which batch was used for this animal?
```

This creates a useful chain:

```text
Vaccine
   ↓
Inventory
   ↓
Batch
   ↓
Vaccination
   ↓
Animal
```

The current finalized Inventory model supports vaccine inventory and inventory transactions, including `Vaccination Usage`. fileciteturn13file6

---

# 24. Important Inventory Consideration

The current `Vaccination` model contains:

```text
batchNumber
```

but the current Inventory model's batch/lot information is not represented as a dedicated batch field in the finalized model.

Therefore, exact batch-level stock reconciliation would require careful implementation.

At minimum:

```text
Vaccination.batchNumber
```

provides event-level traceability.

If future requirements demand strict batch-level inventory accounting, a dedicated inventory-batch design may be considered later.

---

# 25. `status`

### Type

Enum

### Allowed Values

```text
Scheduled
Completed
Missed
Cancelled
```

These are the finalized statuses. fileciteturn13file0

### Purpose

Represents the lifecycle state of the vaccination event.

---

# 26. Status: Scheduled

Meaning:

```text
Vaccination is planned but has not yet been administered.
```

Example:

```text
vaccinationDate = 2026-08-20
status = Scheduled
```

Possible system actions:

```text
Reminder
Upcoming notification
Dashboard display
```

Inventory should normally NOT be consumed yet.

---

# 27. Status: Completed

Meaning:

```text
Animal actually received the vaccine.
```

Example:

```text
vaccinationDate = 2026-08-17
status = Completed
administeredBy = Farm Member A
batchNumber = BATCH-001
```

This is the state that should normally trigger:

```text
Inventory consumption
Next due date
Audit
Completion notification/history
```

---

# 28. Status: Missed

Meaning:

```text
The planned vaccination date passed without successful administration.
```

Example:

```text
vaccinationDate = 2026-08-17
status = Missed
```

Normally:

```text
No vaccination consumption
```

because the vaccine was not necessarily administered.

The system may create a new reminder/rescheduling action.

---

# 29. Status: Cancelled

Meaning:

```text
The scheduled vaccination was intentionally cancelled.
```

Possible reasons can be stored in:

```text
remarks
```

Examples:

```text
Animal unavailable
Vaccine unavailable
Veterinary instruction
Schedule changed
Animal transferred
```

Normally:

```text
No inventory consumption
```

unless the vaccine was actually opened/used and therefore needs a separate wastage transaction.

---

# 30. Recommended Status Transition

Normal lifecycle:

```text
Scheduled
    ↓
Completed
```

or:

```text
Scheduled
    ↓
Missed
```

or:

```text
Scheduled
    ↓
Cancelled
```

A completed vaccination should generally not be changed back to:

```text
Scheduled
```

without an explicit correction workflow.

---

# 31. Invalid Status Transition Example

Bad:

```text
Completed
   ↓
Scheduled
```

This could cause:

```text
Duplicate inventory usage
Incorrect reminders
Incorrect health history
```

Therefore, completed records should be treated as historical events and corrections should be controlled/audited.

---

# 32. `remarks`

### Type

String

### Required

Optional

### Purpose

Stores additional vaccination information.

Examples:

```text
Animal showed mild stress.
Vaccination completed successfully.
Booster postponed due to animal illness.
```

For cancellation/missed events:

```text
Reason for cancellation
Reason for missed vaccination
```

can be stored here.

---

# 33. `createdBy`

### Type

ObjectId / User or FarmMember reference

### Required

Yes

### Purpose

Identifies the user who created the Vaccination record.

This may be different from:

```text
administeredBy
```

---

# 34. `updatedAt`

### Type

Date

### Required

Yes

### Purpose

Stores when the vaccination record was last updated.

---

# 35. Important Observation About `createdAt`

The current finalized Vaccination model contains:

```text
createdBy
updatedAt
```

but does **not** list:

```text
createdAt
```

Therefore, this document does not silently add `createdAt` to the finalized model.

However, from a system-implementation perspective, creation timestamp is useful for audit/history. If the project later wants it, that should be a deliberate schema revision rather than an accidental addition.

---

# 36. Vaccination Creation Workflow

```text
Select Farm
      ↓
Select Animal
      ↓
Verify Animal belongs to Farm
      ↓
Select Vaccine
      ↓
Verify Vaccine exists
      ↓
Verify Vaccine is active
      ↓
Check Animal eligibility
      ↓
Check minimum age
      ↓
Check vaccination history
      ↓
Determine schedule
      ↓
Create Scheduled Vaccination
```

Then:

```text
Scheduled
    ↓
Animal receives vaccine
    ↓
Enter batch number
    ↓
Enter administeredBy
    ↓
Mark Completed
    ↓
Calculate/store nextDueDate
    ↓
Consume inventory
    ↓
Create Inventory_Transaction
    ↓
Create Notification/reminder
    ↓
Create AuditLog
```

---

# 37. Animal Eligibility

Before vaccination, the system should verify:

```text
Animal exists
Animal belongs to farm
Animal is active
Animal is not dead
Animal is not sold
```

and relevant vaccine eligibility:

```text
Minimum age
Previous vaccination history
Configured schedule
```

---

# 38. Animal Lifecycle Validation

If:

```text
lifeStatus = Dead
```

new vaccination should be rejected.

If:

```text
lifeStatus = Sold
```

new vaccination should normally be rejected.

If:

```text
lifeStatus = Transferred
```

the responsible farm and transfer date must determine where the new vaccination should be recorded.

Historical vaccinations remain intact.

---

# 39. Minimum Age Check

Suppose Vaccine:

```text
minimumAge = 4 months
```

Animal:

```text
age = 3 months
```

System:

```text
Warning / reject according to configured policy
```

If:

```text
age = 5 months
```

the minimum-age condition is satisfied.

The exact veterinary eligibility decision should follow the configured vaccination protocol.

---

# 40. Previous Vaccination Check

Before creating a new vaccination, the system should inspect:

```text
Previous Vaccination records
```

for:

```text
same animal
+
same vaccine
```

This helps determine:

```text
Initial dose?
Booster?
Repeat dose?
Already completed?
Overdue?
Duplicate?
```

---

# 41. Duplicate Prevention

Example:

```text
Animal A
FMD Vaccine
2026-08-17
Completed
```

A second identical completion should normally not be created accidentally.

The backend should detect potential duplicate events.

A duplicate rule should consider:

```text
animalId
vaccineId
vaccinationDate
status
```

and the configured vaccination protocol.

Do not blindly impose a permanent unique index on these fields because legitimate booster/repeat vaccinations can use the same vaccine.

---

# 42. Scheduling vs Actual Administration

A good workflow is:

```text
Vaccination planned
       ↓
Scheduled record
       ↓
Actual administration
       ↓
Update same record
       ↓
Completed
```

This is better than creating:

```text
Scheduled record
+
separate Completed record
```

for the same event.

---

# 43. Inventory Consumption Rule

When:

```text
status changes to Completed
```

the system should create:

```text
Inventory_Transaction
```

with:

```text
transactionType = Vaccination Usage
referenceType = Vaccination
referenceId = Vaccination._id
```

The finalized Inventory_Transaction model explicitly supports these values. fileciteturn13file6

---

# 44. Why Inventory Should Not Be Consumed at Scheduling

Suppose:

```text
Scheduled vaccination
```

but the animal is unavailable.

If stock was already reduced:

```text
Inventory = incorrect
```

Therefore:

```text
Scheduled
=
planned

Completed
=
consumed
```

This distinction keeps stock accurate.

---

# 45. Inventory Transaction Example

Before:

```text
FMD vaccine stock = 50 doses
```

Vaccination completed:

```text
1 dose
```

Create:

```text
Inventory_Transaction
transactionType = Vaccination Usage
quantity = 1
referenceType = Vaccination
referenceId = VACC-000001
```

After:

```text
stock = 49 doses
```

The exact unit must match the inventory item's configured unit.

---

# 46. Atomic Completion Workflow

The transition:

```text
Scheduled → Completed
```

should ideally perform these operations atomically:

```text
1. Validate vaccine
2. Validate animal
3. Validate stock
4. Update Vaccination
5. Reduce inventory
6. Create Inventory_Transaction
7. Create AuditLog
8. Create/update reminder
```

If inventory update fails:

```text
Vaccination should not become Completed
```

unless the project has a deliberate exception workflow.

---

# 47. Insufficient Vaccine Stock

Suppose:

```text
Stock = 0
```

but user tries:

```text
Complete vaccination
```

Recommended:

```text
Reject completion
```

with:

```text
Insufficient vaccine stock
```

unless an authorized manual adjustment/override workflow exists.

---

# 48. Vaccine Expiry

Inventory contains:

```text
expiryDate
status
```

according to the finalized Inventory model. fileciteturn13file6

Before vaccination, the system should ideally prevent use of:

```text
Expired
```

vaccine inventory.

This is an Inventory-level validation, while Vaccination records the actual event.

---

# 49. Vaccine Status vs Inventory Status

Do not confuse:

```text
Vaccine.isActive
```

with:

```text
Inventory.status
```

### Vaccine.isActive

```text
Is this vaccine definition available for use?
```

### Inventory.status

```text
What is the current stock/storage condition for this farm's item?
```

Example:

```text
Vaccine.isActive = true

Inventory.status = Expired
```

The vaccine definition is valid, but this particular farm's stock cannot be used.

---

# 50. Next Due Date + Notification

After completion:

```text
nextDueDate
```

can drive:

```text
Notification
```

The finalized Notification model supports:

```text
referenceType = Vaccination
```

and notification types including:

```text
Reminder
Alert
Warning
Information
Success
```

with priorities from Low to Critical. fileciteturn13file17

---

# 51. Upcoming Reminder

Example:

```text
nextDueDate = 2026-09-16
```

The notification service can generate:

```text
Title:
Vaccination Reminder

Message:
FMD vaccination for Animal AN-000125 is due soon.
```

The exact reminder window should be configurable.

---

# 52. Overdue Vaccination

If:

```text
today > nextDueDate
```

and the applicable vaccination has not been completed:

```text
Overdue
```

can be detected by the scheduling service.

No separate `overdue` status is required in the current Vaccination enum.

This avoids duplicating a derived state.

---

# 53. Why No `Overdue` Status?

Current finalized status values are:

```text
Scheduled
Completed
Missed
Cancelled
```

Overdue can be derived from:

```text
status = Scheduled
AND
nextDueDate < today
```

or from the applicable vaccination scheduling workflow.

This keeps the enum smaller.

---

# 54. Missed Vaccination Workflow

Example:

```text
Scheduled:
2026-08-17

Today:
2026-08-18

No completion
```

The system can mark:

```text
status = Missed
```

and then:

```text
Create reminder/reschedule task
```

No inventory consumption should occur unless vaccine was actually used.

---

# 55. Cancelled Vaccination Workflow

Example:

```text
Scheduled
   ↓
Animal transferred
   ↓
Cancelled
```

Store reason in:

```text
remarks
```

and preserve the historical event.

---

# 56. Completed Vaccination Correction

Suppose:

```text
Batch number entered incorrectly
```

Original:

```text
BATCH-001
```

Correct:

```text
BATCH-002
```

The system should:

```text
Update Vaccination
+
Audit old/new values
```

If inventory batch tracking is implemented, inventory reconciliation must also be considered.

---

# 57. Completed Vaccination Should Not Be Silently Deleted

Once:

```text
Completed
```

the event may already have caused:

```text
Inventory consumption
Notification
Reports
Compliance status
Audit
```

Therefore, hard deletion is dangerous.

Prefer:

```text
Controlled correction
```

or a formal void/reversal workflow if the business requires it.

---

# 58. Vaccination + Disease

The Vaccination record does not directly contain:

```text
diseaseId
```

because the selected Vaccine already contains:

```text
targetDiseaseIds[]
```

Therefore:

```text
Vaccination
   ↓
vaccineId
   ↓
Vaccine
   ↓
targetDiseaseIds[]
```

This avoids unnecessary duplication.

---

# 59. Vaccination + Treatment

Vaccination and Treatment are separate health events.

```text
Vaccination
=
Preventive health action

Treatment
=
Response to disease/health condition
```

They can both appear in the animal's health timeline.

---

# 60. Vaccination + Disease Prevention

The Disease model supports:

```text
prevention
recommendedVaccineIds[]
```

Therefore:

```text
Disease
   ↓
Recommended Vaccine
   ↓
Vaccination
   ↓
Animal protection history
```

This creates a coherent prevention workflow.

---

# 61. Vaccination + AuditLog

Important actions to audit:

```text
Vaccination Created
Vaccination Scheduled
Vaccination Completed
Vaccination Missed
Vaccination Cancelled
Vaccination Updated
Batch Changed
Administrator Changed
Vaccination Date Changed
Next Due Date Changed
Vaccine Changed
```

The finalized AuditLog model is designed to record entity/action changes with old/new data and changed fields.

---

# 62. Vaccination + Notification

Potential notifications:

```text
Upcoming vaccination
Vaccination due today
Vaccination overdue
Mandatory vaccine overdue
Vaccination completed
Vaccination missed
```

These are derived application events.

---

# 63. Vaccination + Reports

The finalized Report model supports:

```text
reportType = Vaccination
```

Therefore, Vaccination records can power:

```text
Vaccination History Report
Vaccination Compliance Report
Overdue Vaccination Report
Vaccine Usage Report
Animal Vaccination Report
```

The Report model supports PDF, Excel, CSV and JSON formats. fileciteturn13file17

---

# 64. Vaccination Compliance

For mandatory vaccines:

```text
isMandatory = true
```

the system can calculate:

```text
Completed
Scheduled
Missed
Overdue
```

at farm level.

Example:

```text
Mandatory vaccinations due = 100
Completed = 85
Pending = 10
Missed = 5
```

This is an analytics/reporting calculation.

---

# 65. Vaccination History for Animal

Animal profile can display:

```text
Vaccine
Vaccination Date
Next Due Date
Batch
Administered By
Status
Remarks
```

Example:

```text
FMD Vaccine
17 Aug 2026
Next: 16 Sep 2026
Batch: FMD-001
Status: Completed
```

---

# 66. Current vs Historical Vaccine Definition

Suppose:

```text
Vaccine V001
```

is edited after an old vaccination.

The old Vaccination still references:

```text
V001
```

Therefore, historical master-data changes should be carefully audited.

The current finalized model does not contain vaccine field snapshots such as:

```text
administeredDosage
administeredRoute
manufacturerSnapshot
```

So the current implementation should rely on controlled master-data changes and AuditLog.

If exact historical product configuration becomes a thesis requirement later, a snapshot/versioning enhancement can be considered separately.

---

# 67. Important `createdBy` Logic

`createdBy` answers:

```text
Who created this database event?
```

`administeredBy` answers:

```text
Who actually administered the vaccine?
```

Example:

```text
Administered By:
Farm Worker A

Created By:
Farm Manager B
```

Both are useful and should not be merged into one field.

---

# 68. Authorization

Recommended permissions:

### Farm Owner / Manager

Can:

```text
Schedule
Complete
Update
View
```

according to the final role permission matrix.

### Worker

Can:

```text
View
Record vaccination
```

if authorized.

### Unauthorized user

Cannot create or modify another farm's vaccination records.

The backend must validate farm membership/role.

---

# 69. Vaccination Date Validation

For:

```text
Completed
```

recommended:

```text
vaccinationDate <= current date
```

because an actual completed vaccination cannot normally occur in the future.

For:

```text
Scheduled
```

future dates are valid.

This distinction is important.

---

# 70. Backdated Vaccination

A farmer may enter a completed vaccination later.

Example:

```text
Actual vaccination:
Aug 15

Record entered:
Aug 17
```

Then:

```text
vaccinationDate = Aug 15
updatedAt = Aug 17
```

AuditLog records the later entry/change.

---

# 71. Future-Dated Completed Vaccination

Input:

```text
vaccinationDate = tomorrow
status = Completed
```

Recommended:

```text
Reject
```

or require a special authorized workflow.

The ordinary Completed state should represent an already-administered event.

---

# 72. Batch Number Validation

For:

```text
Completed
```

recommended:

```text
batchNumber required
```

because traceability is most valuable for actual administration.

For:

```text
Scheduled
```

batch number may be unknown until the actual dose is selected.

Therefore:

```text
batchNumber = optional before completion
batchNumber = required on completion
```

is a reasonable business rule.

---

# 73. AdministeredBy Validation

For:

```text
Completed
```

recommended:

```text
administeredBy required
```

For:

```text
Scheduled
```

it can remain null.

This matches the difference between planned and completed events.

---

# 74. Status Transition Table

| Current | Next | Allowed? | Reason |
|---|---|---|---|
| Scheduled | Completed | Yes | Normal completion |
| Scheduled | Missed | Yes | Date passed without administration |
| Scheduled | Cancelled | Yes | Planned event cancelled |
| Completed | Scheduled | No normally | Would corrupt history |
| Completed | Missed | No normally | Contradictory |
| Completed | Cancelled | No normally | Requires formal correction |
| Missed | Scheduled | Conditional | Rescheduling may create/update a new planned event |
| Missed | Completed | Conditional | Only if correcting data, not normal flow |
| Cancelled | Scheduled | Conditional | New scheduling should generally be a new event |
| Cancelled | Completed | No normally | Contradictory |

---

# 75. Why Rescheduling May Create a New Record

Suppose:

```text
Scheduled:
Aug 17
```

was missed.

Instead of changing the old event repeatedly:

```text
Aug 17 → Aug 20 → Aug 25
```

a cleaner historical approach can be:

```text
VACC-001
Aug 17
Missed

VACC-002
Aug 25
Scheduled
```

This preserves the actual schedule history.

The exact rescheduling UX can be decided by the implementation.

---

# 76. Inventory Reconciliation on Correction

Suppose:

```text
Vaccination V1
Completed
```

and inventory consumed:

```text
1 dose
```

If the vaccination is later voided/corrected, inventory must not remain incorrectly reduced.

The correction workflow should perform:

```text
Vaccination correction
+
Inventory reversal/adjustment
+
AuditLog
```

atomically where possible.

---

# 77. Vaccine Usage Traceability

The ideal traceability chain is:

```text
Vaccine Master
      ↓
Inventory
      ↓
Inventory Transaction
      ↓
Vaccination
      ↓
Animal
```

and reverse:

```text
Animal
   ↓
Vaccination
   ↓
Vaccine
   ↓
Batch Number
```

This is useful for audits and operational reports.

---

# 78. Performance / Index Recommendations

Useful indexes:

```javascript
farmId: 1
animalId: 1
vaccineId: 1
vaccinationDate: -1
nextDueDate: 1
status: 1
batchNumber: 1
administeredBy: 1
```

Highly useful compound index:

```javascript
{
    farmId: 1,
    animalId: 1,
    vaccinationDate: -1
}
```

For reminder queries:

```javascript
{
    farmId: 1,
    status: 1,
    nextDueDate: 1
}
```

These support:

```text
Animal vaccination history
Upcoming vaccinations
Overdue vaccinations
Farm vaccination dashboard
```

---

# 79. Reminder Query Concept

Conceptually:

```text
Find vaccinations where:

status = Scheduled
AND
nextDueDate <= reminderThreshold
AND
nextDueDate >= today
```

Then:

```text
Create Notification
```

The exact reminder threshold should be configurable.

---

# 80. Overdue Query Concept

Conceptually:

```text
status = Scheduled
AND
nextDueDate < today
```

Then:

```text
Overdue warning
```

The system may choose to mark the event:

```text
Missed
```

after an appropriate business condition, rather than automatically changing it immediately.

---

# 81. Example — Complete Vaccination

```javascript
{
    vaccinationNo: "VACC-000001",

    farmId: "FARM-001",

    animalId: "AN-000125",

    vaccineId: "VAC-FMD-001",

    vaccinationDate: "2026-08-17",

    nextDueDate: "2026-09-16",

    administeredBy: "FARMMEMBER-001",

    batchNumber: "FMD-BATCH-2026-08",

    status: "Completed",

    remarks: "Vaccination completed successfully.",

    createdBy: "FARMMEMBER-002",

    updatedAt: "2026-08-17T10:30:00Z"
}
```

Values are illustrative.

---

# 82. Example — Scheduled Vaccination

```javascript
{
    vaccinationNo: "VACC-000002",

    farmId: "FARM-001",

    animalId: "AN-000126",

    vaccineId: "VAC-FMD-001",

    vaccinationDate: "2026-08-25",

    nextDueDate: null,

    administeredBy: null,

    batchNumber: null,

    status: "Scheduled",

    remarks: "Scheduled according to vaccination plan.",

    createdBy: "FARMMEMBER-002",

    updatedAt: "2026-08-17T10:40:00Z"
}
```

---

# 83. Example — Missed Vaccination

```javascript
{
    vaccinationNo: "VACC-000003",

    farmId: "FARM-001",

    animalId: "AN-000127",

    vaccineId: "VAC-FMD-001",

    vaccinationDate: "2026-08-10",

    nextDueDate: null,

    administeredBy: null,

    batchNumber: null,

    status: "Missed",

    remarks: "Animal was unavailable on scheduled date.",

    createdBy: "FARMMEMBER-002",

    updatedAt: "2026-08-17T11:00:00Z"
}
```

---

# 84. Example — Cancelled Vaccination

```javascript
{
    vaccinationNo: "VACC-000004",

    farmId: "FARM-001",

    animalId: "AN-000128",

    vaccineId: "VAC-FMD-001",

    vaccinationDate: "2026-08-20",

    nextDueDate: null,

    administeredBy: null,

    batchNumber: null,

    status: "Cancelled",

    remarks: "Vaccination cancelled because animal was transferred.",

    createdBy: "FARMMEMBER-002",

    updatedAt: "2026-08-17T11:15:00Z"
}
```

---

# 85. Error Scenario — Inactive Vaccine

User tries:

```text
vaccineId = V001
```

but:

```text
Vaccine.isActive = false
```

Recommended:

```text
Reject new routine vaccination
```

Historical records remain accessible.

---

# 86. Error Scenario — Animal from Another Farm

Input:

```text
farmId = Farm A
animalId = Animal from Farm B
```

Backend:

```text
Reject
```

---

# 87. Error Scenario — No Stock

```text
Vaccine stock = 0
```

User tries:

```text
Scheduled → Completed
```

Recommended:

```text
Reject completion
```

unless authorized override is implemented.

---

# 88. Error Scenario — Completed Without Administrator

Input:

```text
status = Completed
administeredBy = null
```

Recommended:

```text
Validation error
```

because a completed vaccination should have an accountable administrator.

---

# 89. Error Scenario — Completed Without Batch

If the project adopts batch traceability as required for completed events:

```text
status = Completed
batchNumber = null
```

should be rejected.

---

# 90. Error Scenario — Future Completed Date

```text
vaccinationDate = tomorrow
status = Completed
```

Reject or require special correction workflow.

---

# 91. Error Scenario — Duplicate Completion

Existing:

```text
Animal A
FMD
Aug 17
Completed
```

New:

```text
Animal A
FMD
Aug 17
Completed
```

The system should detect it as a potential duplicate.

Do not blindly reject every same-vaccine record forever because legitimate booster/repeat events are possible.

---

# 92. Vaccination History

Animal profile can show:

```text
Vaccination History
----------------------------------------
Vaccine     Date       Status
FMD         Aug 17     Completed
HS          Aug 20     Completed
FMD         Sep 16     Scheduled
```

This provides a clear health timeline.

---

# 93. Vaccination Compliance Dashboard

Farm-level dashboard:

```text
Total Scheduled
Total Completed
Total Missed
Total Cancelled
Upcoming
Overdue
Mandatory Compliance
```

Example:

```text
Completed: 82%
Scheduled: 10%
Missed: 5%
Cancelled: 3%
```

These are derived analytics.

---

# 94. Vaccine Coverage

For a selected vaccine:

```text
Eligible animals
        ↓
Completed vaccinations
        ↓
Coverage %
```

Conceptually:

```text
Coverage =
vaccinated eligible animals
÷
total eligible animals
× 100
```

Eligibility rules must be defined by the vaccination program.

---

# 95. Vaccination and Animal Health Timeline

The animal's health timeline can combine:

```text
Vaccination
Treatment
Disease
Weight
Milk
Breeding
Pregnancy
```

Example:

```text
Aug 10 → Weight measurement
Aug 12 → Treatment
Aug 17 → FMD Vaccination
Aug 20 → Milk record
```

This gives the farmer a unified animal history.

---

# 96. Vaccination and Treatment Interaction

A vaccination is preventive, while Treatment addresses an existing health issue.

The system may display warnings when an animal is currently:

```text
Under Treatment
```

but should not automatically block vaccination unless the configured veterinary policy requires it.

This should remain an application/business rule rather than an assumption in the schema.

---

# 97. Vaccination and Notification

Notification can be generated for:

```text
Scheduled
Upcoming
Due Today
Overdue
Missed
Completed
```

But the notification is a separate entity.

```text
Vaccination
   ↓
Notification
```

The finalized Notification model supports `referenceType = Vaccination` and `referenceId` for this purpose. fileciteturn13file17

---

# 98. Vaccination and Report

The Report module can generate:

```text
Vaccination History
Vaccination Compliance
Overdue Vaccinations
Vaccine Usage
Animal Health Report
```

The current finalized Report model explicitly supports `Vaccination` as a report type. fileciteturn13file17

---

# 99. Audit Strategy

Vaccination is a high-value historical event because it can affect:

```text
Animal health
Inventory
Compliance
Notifications
Reports
```

Therefore, important changes should always be auditable.

At minimum:

```text
Create
Update
Complete
Miss
Cancel
Correct
```

should generate appropriate AuditLog events.

---

# 100. Recommended Service-Layer Responsibilities

Instead of putting all business logic inside the Mongoose schema, use a service layer.

Example:

```text
vaccinationService.schedule()
vaccinationService.complete()
vaccinationService.markMissed()
vaccinationService.cancel()
vaccinationService.correct()
vaccinationService.calculateNextDueDate()
vaccinationService.consumeInventory()
```

This is especially useful because completing a vaccination affects multiple collections.

---

# 101. Why Service Layer Is Important

Completing a vaccination may require:

```text
Vaccination update
+
Inventory update
+
Inventory transaction
+
Notification
+
AuditLog
```

This is more than a simple CRUD operation.

Therefore, a dedicated business service is cleaner and safer than putting all logic inside a controller.

---

# 102. Transaction Boundary

Recommended transaction boundary for completion:

```text
BEGIN TRANSACTION

Validate
   ↓
Update Vaccination
   ↓
Update Inventory
   ↓
Create Inventory_Transaction
   ↓
Create AuditLog

COMMIT
```

If a critical database operation fails:

```text
ROLLBACK
```

Notifications may be handled after successful commit depending on implementation.

---

# 103. Historical Integrity

Once:

```text
status = Completed
```

the record should be treated as historical.

Do not casually change:

```text
animalId
vaccineId
vaccinationDate
```

after downstream records exist.

If correction is necessary:

```text
Controlled correction
+
AuditLog
+
Inventory reconciliation
```

---

# 104. Final Business Rules Checklist

```text
[✓] Vaccination is an animal-level event/history record
[✓] Vaccine is the master definition
[✓] vaccineId references Vaccine._id
[✓] vaccinationNo should be unique
[✓] farmId is required
[✓] animalId is required
[✓] Animal must belong to farmId
[✓] Vaccine must exist
[✓] New routine vaccination should use an active Vaccine
[✓] vaccinationDate has different meaning for Scheduled vs Completed
[✓] Completed means vaccine was actually administered
[✓] Scheduled means planned, not administered
[✓] Missed means planned but not completed
[✓] Cancelled means intentionally cancelled
[✓] nextDueDate is animal/event-specific
[✓] nextDueDate is derived using Vaccine scheduling rules
[✓] administeredBy is required for Completed
[✓] batchNumber is recommended/required for Completed under traceability policy
[✓] createdBy identifies who created the record
[✓] administeredBy identifies who administered the vaccine
[✓] Historical events should not be casually deleted
[✓] Completed → Scheduled should normally be blocked
[✓] Scheduled → Completed is the normal flow
[✓] Inventory is normally consumed only on Completed
[✓] Vaccination Usage creates Inventory_Transaction
[✓] Insufficient stock should normally block completion
[✓] Expired vaccine stock should not be used
[✓] Upcoming vaccination can generate Notification
[✓] Overdue can be derived from due date/status
[✓] Mandatory vaccines can support compliance analytics
[✓] Disease relationship comes through Vaccine.targetDiseaseIds
[✓] Vaccination can be reported through Report
[✓] Important changes should generate AuditLog
[✓] Multi-collection completion should use transaction/service logic
[✓] Backdated completed vaccination can be supported
[✓] Future-dated Completed vaccination should normally be rejected
[✓] Duplicate vaccination events should be detected
```

---

# 105. Final Status

**Model:** `Vaccination`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Animal-level Vaccination History:** ✅  
**Scheduling:** ✅  
**Booster/Repeat Support:** ✅  
**Inventory Integration:** ✅  
**Notification Integration:** ✅  
**Audit Integration:** ✅  
**Historical Integrity:** ✅  
**Role/Authorization Support:** ✅  

**Next Model:** `Disease`
