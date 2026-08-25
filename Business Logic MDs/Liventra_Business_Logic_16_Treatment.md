# Liventra — Business Logic Documentation
## Model 16: Treatment

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** Health & Treatment

---

# 1. Source / Finalized Model

The current finalized Liventra DB model defines `Treatment` as:

```text
Treatment Model:
_id
treatmentNo
farmId
animalId
diseaseId
diagnosisDate
symptoms[]
treatedBy
medicines:[
    {
        medicineId
        dosage
        dosageUnit
        frequency
        duration
        administrationRoute
        instructions
    }
]
estimatedCost
status: Ongoing | Recovered | Referred | Cancelled
followUpDate
completedDate
remarks
createdBy
createdAt
updatedAt
```

This exact Treatment structure is present in the finalized DB-model source. fileciteturn14file1

> **Important:** The finalized model currently uses **`diseaseId` (singular)**, not `diseaseIds[]`. This document follows the finalized source exactly rather than silently changing it.

---

# 2. Purpose

`Treatment` records an **animal-specific health treatment event**.

It answers:

> "What disease/condition was diagnosed in this animal, when was it diagnosed, what symptoms were recorded, which medicines were used, who handled the treatment, what was the estimated cost, what is the current treatment status, and when should the animal be followed up?"

Example:

```text
Animal:
AN-000125

Disease:
D001

Diagnosis:
2026-08-17

Symptoms:
Fever
Reduced appetite

Medicines:
Medicine A
Medicine B

Status:
Ongoing

Follow-up:
2026-08-22
```

---

# 3. Treatment vs Disease

This distinction is fundamental.

## Disease

```text
What is the disease?
```

It is master/reference data.

Example:

```text
Disease:
Foot and Mouth Disease
Category:
Viral
Symptoms:
...
Prevention:
...
```

## Treatment

```text
What happened to this particular animal?
```

Example:

```text
Animal A
Disease D001
Diagnosis Date
Symptoms
Medicines
Follow-up
Status
```

Therefore:

```text
Disease
   ↓
MASTER DATA

Treatment
   ↓
ANIMAL-SPECIFIC EVENT
```

---

# 4. Treatment vs Medicine

Similarly:

## Medicine

```text
What medicine/product is this?
```

Contains:

```text
Medicine identity
Generic name
Brand
Manufacturer
Category
Dosage form
Strength
Route
Withdrawal period
```

## Treatment

```text
How was that medicine used for this animal?
```

Contains:

```text
Medicine ID
Dosage
Dosage unit
Frequency
Duration
Route
Instructions
```

Therefore:

```text
Medicine
   ↓
MASTER PRODUCT

Treatment.medicines[]
   ↓
ACTUAL TREATMENT INSTRUCTION
```

---

# 5. Overall Health Architecture

The core health flow is:

```text
Disease
   ↓
Treatment
   ↓
Medicine(s)
   ↓
Animal
```

Operationally:

```text
Animal
   ↓
Diagnosis
   ↓
Treatment
   ↓
Medicine
   ↓
Inventory
   ↓
Treatment Usage
   ↓
Withdrawal Period
   ↓
Follow-up
   ↓
Recovery / Referral / Cancellation
```

Vaccination is a separate preventive-health workflow:

```text
Disease
   ↓
Vaccine
   ↓
Vaccination
```

So:

```text
Treatment
=
Curative / clinical management event

Vaccination
=
Preventive immunization event
```

---

# 6. Why Treatment Needs Its Own Collection

Without Treatment, Liventra would only know:

```text
Disease exists
Medicine exists
```

but not:

```text
Which animal was treated?
When?
Why?
With what medicine?
What happened afterward?
```

Treatment connects the master data to real farm activity.

---

# 7. Field-by-Field Business Logic

# 7.1 `_id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique database identifier of the treatment record.

Example:

```text
ObjectId(...)
```

This is different from:

```text
treatmentNo
```

---

# 7.2 `treatmentNo`

### Type

String

### Required

Yes

### Example

```text
TRT-000001
```

### Purpose

Human-readable unique treatment identifier.

Used in:

```text
Search
Reports
Animal history
AuditLog
Support
```

---

# 8. Treatment Number Generation

Recommended:

```text
TRT-{SEQUENCE}
```

Examples:

```text
TRT-000001
TRT-000002
TRT-000003
```

The backend should generate/validate uniqueness.

The frontend should not be trusted to generate unique treatment numbers.

---

# 9. `farmId`

### Type

ObjectId / Farm reference

### Required

Yes

### Purpose

Identifies the farm where the treatment belongs.

This is essential for multi-farm data isolation.

---

# 10. Farm Isolation Rule

Before creating or updating a Treatment:

```text
Treatment.farmId
        ↓
must match
        ↓
Animal.farmId
```

Example:

```text
Treatment.farmId = Farm A
Animal.farmId = Farm B
```

must be rejected.

This protects against cross-farm data access.

The SRS explicitly requires that users must not access another farm's data. fileciteturn14file19

---

# 11. `animalId`

### Type

ObjectId / Animal reference

### Required

Yes

### Purpose

Identifies the animal receiving treatment.

One animal can have multiple Treatment records:

```text
Animal A
   |
   +--- Treatment 1
   +--- Treatment 2
   +--- Treatment 3
```

This creates the animal's treatment history.

---

# 12. Animal Validation

Before creating a Treatment:

```text
Animal exists
AND
Animal belongs to farmId
```

The system should also verify the animal's lifecycle.

Normally, new treatment should not be created for:

```text
lifeStatus = Dead
lifeStatus = Sold
```

unless an explicit historical correction/import workflow exists.

---

# 13. `diseaseId`

### Type

ObjectId / Disease reference

### Required

Yes

### Purpose

Identifies the disease/condition associated with this treatment.

Example:

```text
diseaseId → DIS-FMD-001
```

---

# 14. Important: `diseaseId` Is Singular

The finalized model explicitly uses:

```text
diseaseId
```

not:

```text
diseaseIds[]
```

Therefore, the current Treatment design represents:

```text
One Treatment
     ↓
One primary Disease
```

If multiple conditions ever need to be associated with one clinical event, that would require a future schema decision.

For the current finalized model, do not silently convert it to an array.

---

# 15. Disease Reference Integrity

The backend should verify:

```text
Disease exists
```

before creating Treatment.

If:

```text
diseaseId = nonexistent ID
```

then:

```text
Reject
```

---

# 16. Disease Active/Inactive Logic

If:

```text
Disease.isActive = false
```

it should normally not be selectable for a new routine Treatment.

However, historical Treatment records referencing that disease must remain valid.

Therefore:

```text
Disease deactivated
       ↓
New treatment selection blocked
       ↓
Old treatment history preserved
```

---

# 17. `diagnosisDate`

### Type

Date

### Required

Yes

### Purpose

Stores the date on which the disease/condition was diagnosed or recorded.

Example:

```text
2026-08-17
```

---

# 18. Diagnosis Date vs Created At

These are different concepts.

Example:

```text
Animal diagnosed:
Aug 15

Record entered:
Aug 17
```

Then:

```text
diagnosisDate = Aug 15
createdAt = Aug 17
```

This preserves the real clinical timeline.

---

# 19. Backdated Treatment Entry

A farmer may enter a treatment later than the actual diagnosis.

Therefore:

```text
diagnosisDate
```

can be earlier than:

```text
createdAt
```

This should be allowed where appropriate.

The difference should remain auditable.

---

# 20. Future Diagnosis Date

A normal treatment record should generally reject:

```text
diagnosisDate > current date
```

unless the system explicitly supports scheduled/planned diagnosis-related entries.

The current Treatment status does not include a separate `Scheduled` state, so future diagnosis dates should normally be rejected.

---

# 21. `symptoms[]`

### Type

Array of strings

### Required

Recommended

### Example

```text
[
    "Fever",
    "Reduced appetite",
    "Excessive salivation"
]
```

### Purpose

Stores symptoms observed/reported during diagnosis.

---

# 22. Why Symptoms Are Stored in Treatment

Disease has master symptoms:

```text
Disease.symptoms[]
```

But the actual animal may present only some of those symptoms.

Therefore:

```text
Disease.symptoms[]
=
General/reference symptoms

Treatment.symptoms[]
=
Symptoms observed in this animal/event
```

This is an important distinction.

---

# 23. Example

Disease master:

```text
FMD

Symptoms:
[
    Fever,
    Salivation,
    Mouth lesions,
    Lameness
]
```

Animal A Treatment:

```text
symptoms:
[
    Fever,
    Salivation
]
```

This means Animal A showed those observed symptoms during this treatment event.

---

# 24. `treatedBy`

### Type

ObjectId / authorized User or FarmMember reference

### Required

Yes

### Purpose

Identifies the person responsible for/administering the treatment according to the application's workflow.

---

# 25. No Separate Doctor Model

The current architecture does **not** contain a Doctor collection.

Therefore, do not create:

```text
doctorId
```

for Treatment.

Instead:

```text
treatedBy
```

should reference the authorized user/farm-member identity used by Liventra.

This follows the finalized Treatment model. fileciteturn14file1

---

# 26. `treatedBy` vs `createdBy`

These can be different people.

Example:

```text
Worker A
→ handles treatment

Manager B
→ enters/creates the Treatment record
```

Then:

```text
treatedBy = Worker A
createdBy = Manager B
```

This improves accountability.

---

# 27. `medicines[]`

### Type

Array of nested objects

### Required

Optional depending on treatment type, but generally required for medication-based treatment

### Purpose

Stores the medicine instructions for this specific treatment event.

Structure:

```text
medicines:[
    {
        medicineId,
        dosage,
        dosageUnit,
        frequency,
        duration,
        administrationRoute,
        instructions
    }
]
```

---

# 28. Why `medicines[]` Is an Array

One treatment can require multiple medicines.

Example:

```text
Treatment
   |
   +--- Medicine A
   +--- Medicine B
   +--- Medicine C
```

This avoids creating separate Treatment records for every medicine used in the same clinical event.

---

# 29. Nested `medicineId`

### Type

ObjectId / Medicine reference

### Required

Yes inside each medicine object

### Purpose

Identifies which Medicine master record is being used.

Example:

```text
medicineId → MED-ANT-001
```

---

# 30. Medicine Reference Validation

For each:

```text
medicines[].medicineId
```

the backend should verify:

```text
Medicine exists
```

and normally:

```text
Medicine.isActive = true
```

for new treatment entry.

Historical Treatment records can continue referencing inactive medicines.

---

# 31. `dosage`

### Type

Number

### Required

Yes for medication-based treatment

### Example

```text
5
```

If:

```text
dosageUnit = ml
```

then:

```text
5 ml
```

### Purpose

Stores the animal-specific dose used in this Treatment.

---

# 32. Medicine Strength vs Treatment Dosage

This distinction is critical.

Medicine:

```text
strength = 100 mg/ml
```

Treatment:

```text
dosage = 5
dosageUnit = ml
```

Medicine answers:

```text
What is the product concentration?
```

Treatment answers:

```text
How much was used for this animal?
```

---

# 33. `dosageUnit`

### Type

Enum/string according to implementation

### Example

```text
ml
mg
tablet
dose
```

The unit should be compatible with the selected Medicine/dosage form.

Example:

```text
Injection
→ ml
```

where appropriate.

---

# 34. Dosage Validation

Recommended:

```text
dosage > 0
```

Reject:

```text
dosage = 0
dosage = -5
```

for medication administration.

The exact acceptable unit/dose must come from the authorized treatment plan.

---

# 35. `frequency`

### Type

String

### Required

Yes for recurring medication

### Examples

```text
Once daily
Twice daily
Every 12 hours
Every 8 hours
As directed
```

### Purpose

Defines how frequently the medicine is to be administered.

---

# 36. Frequency as String

The current finalized model defines:

```text
frequency
```

without a structured numeric interval field.

Therefore, implementation should preserve the finalized model.

If later the system needs automated dose scheduling, a structured representation such as:

```text
frequencyValue
frequencyUnit
```

could be considered as a future schema enhancement.

---

# 37. `duration`

### Type

String

### Required

Yes for time-bound medication

### Examples

```text
5 days
7 days
Until follow-up
As directed
```

### Purpose

Defines how long the medicine is intended to be used.

---

# 38. Duration + Withdrawal Logic

The final applicable administration date is important for calculating withdrawal.

Conceptually:

```text
Treatment duration
      ↓
Last applicable medicine administration
      ↓
Medicine.withdrawalPeriodDays
      ↓
Withdrawal end
```

The current Treatment model does not store a separate:

```text
lastAdministrationDate
```

so implementation must derive it from the treatment schedule or future administration workflow.

---

# 39. `administrationRoute`

### Type

Enum

### Allowed values should align with Medicine:

```text
IM
IV
SC
Oral
Topical
Intranasal
```

### Purpose

Stores the route used for this particular treatment medicine entry.

---

# 40. Medicine Route vs Treatment Route

Medicine master:

```text
default/allowed route
```

Treatment:

```text
route used/recorded in this treatment
```

Example:

```text
Medicine:
administrationRoute = IM

Treatment:
administrationRoute = IM
```

Backend should validate compatibility.

---

# 41. Route/Form Validation

Example:

```text
Medicine dosageForm = Ointment
Treatment administrationRoute = IV
```

should be rejected.

The route must be consistent with the Medicine configuration and authorized treatment plan.

---

# 42. `instructions`

### Type

String

### Required

Optional

### Example

```text
Administer after feeding.
Follow the authorized treatment plan.
```

### Purpose

Stores treatment-specific instructions for this medicine.

This is different from the Medicine master's general description.

---

# 43. Medicine Description vs Treatment Instructions

Medicine:

```text
description
=
general product information
```

Treatment:

```text
instructions
=
animal/treatment-specific instruction
```

Example:

```text
Medicine.description:
General product information.

Treatment.instructions:
Administer according to the approved treatment plan.
```

---

# 44. `estimatedCost`

### Type

Number

### Required

Optional/recommended

### Example

```text
850
```

### Purpose

Stores the estimated total cost of the treatment.

This may include:

```text
Medicine cost
Veterinary/clinical service cost
Other treatment-related cost
```

depending on the project's cost policy.

---

# 45. Estimated Cost vs Actual Cost

The current model explicitly says:

```text
estimatedCost
```

not:

```text
actualCost
```

Therefore, do not treat it as a guaranteed final expense.

Actual financial transactions should be handled through:

```text
Expense
Inventory_Transaction
```

where applicable.

---

# 46. Treatment Cost and Inventory

Medicine consumption can generate:

```text
Inventory_Transaction
```

and inventory purchase costs can be tracked separately.

Therefore:

```text
estimatedCost
=
Treatment-level estimate

Expense
=
Financial transaction

Inventory_Transaction
=
Stock movement/cost record
```

These should not be mixed.

---

# 47. `status`

### Type

Enum

### Allowed Values

```text
Ongoing
Recovered
Referred
Cancelled
```

These are the finalized Treatment statuses. fileciteturn14file1

---

# 48. Status: Ongoing

Meaning:

```text
Treatment is currently active/in progress.
```

Example:

```text
Treatment started
Medicine course ongoing
Follow-up pending
```

Animal may remain:

```text
healthStatus = Under Treatment
```

depending on the animal-health workflow.

---

# 49. Status: Recovered

Meaning:

```text
Treatment has been completed successfully and the animal is considered recovered.
```

Recommended requirements:

```text
completedDate
```

should normally be populated.

---

# 50. Status: Referred

Meaning:

```text
The case has been referred for higher/specialized veterinary care.
```

Example:

```text
Farm-level management insufficient
→ referred to external veterinary service
```

The exact referral details are not represented by a dedicated field in the current model, so relevant information can be captured in:

```text
remarks
```

unless a future referral model is introduced.

---

# 51. Status: Cancelled

Meaning:

```text
Treatment record/process was cancelled.
```

Possible reasons:

```text
Duplicate entry
Wrong animal
Wrong disease
Animal transferred
Treatment plan changed
```

Reason should be recorded in:

```text
remarks
```

---

# 52. Status Transition

Normal flow:

```text
Ongoing
   ↓
Recovered
```

Alternative:

```text
Ongoing
   ↓
Referred
```

or:

```text
Ongoing
   ↓
Cancelled
```

A `Recovered` treatment should not normally go back to `Ongoing` without a controlled correction/reopen workflow.

---

# 53. Status Transition Table

| Current | Next | Normal? |
|---|---|---|
| Ongoing | Recovered | Yes |
| Ongoing | Referred | Yes |
| Ongoing | Cancelled | Yes |
| Recovered | Ongoing | No normally |
| Recovered | Cancelled | No normally |
| Referred | Ongoing | Conditional |
| Referred | Recovered | Conditional |
| Cancelled | Ongoing | Conditional/new treatment preferred |
| Cancelled | Recovered | No normally |

The exact correction workflow should be controlled and audited.

---

# 54. `followUpDate`

### Type

Date

### Required

Optional/conditional

### Purpose

Stores the date when the animal should be reassessed.

Example:

```text
followUpDate = 2026-08-22
```

---

# 55. Follow-up Logic

If:

```text
status = Ongoing
```

and:

```text
followUpDate = future date
```

the system can generate:

```text
Follow-up Reminder
```

This matches the SRS's Notification requirement for health-related reminders such as doctor visit/follow-up. fileciteturn14file19

---

# 56. Follow-up Reminder

Example:

```text
Treatment:
TRT-000001

Animal:
AN-000125

Follow-up:
2026-08-22
```

Notification:

```text
Treatment Follow-up Reminder
Animal AN-000125 requires follow-up.
```

The exact reminder window should be configurable.

---

# 57. Follow-up Date Validation

If:

```text
followUpDate < diagnosisDate
```

the system should normally reject it unless it represents a historical correction.

If:

```text
status = Recovered
```

a future follow-up may still be possible, but the business purpose should be explicit.

---

# 58. `completedDate`

### Type

Date

### Required

Conditional

### Purpose

Stores when the treatment was completed.

Normally required when:

```text
status = Recovered
```

---

# 59. Completed Date Rules

For:

```text
status = Recovered
```

recommended:

```text
completedDate required
```

For:

```text
status = Ongoing
```

normally:

```text
completedDate = null
```

For:

```text
status = Cancelled
```

normally no completed date.

---

# 60. Diagnosis Date vs Completed Date

Example:

```text
diagnosisDate:
Aug 17

completedDate:
Aug 22
```

This gives:

```text
Treatment duration = 5 days
```

at a high level.

This is useful for:

```text
Health reports
Treatment analytics
Recovery analysis
```

---

# 61. `remarks`

### Type

String

### Required

Optional

### Purpose

Stores additional treatment information.

Examples:

```text
Animal responded well to treatment.
```

or:

```text
Referred to external veterinary service due to worsening condition.
```

or:

```text
Cancelled because duplicate treatment record was created.
```

---

# 62. `createdBy`

### Type

ObjectId / User or FarmMember reference

### Required

Yes

### Purpose

Identifies the user who created the Treatment record.

---

# 63. `createdAt`

### Type

Date

### Required

Automatic

### Purpose

Stores when the Treatment record was created.

---

# 64. `updatedAt`

### Type

Date

### Required

Automatic

### Purpose

Stores when the Treatment record was last updated.

---

# 65. Treatment Creation Workflow

Recommended flow:

```text
Select Farm
      ↓
Select Animal
      ↓
Validate animal belongs to farm
      ↓
Select Disease
      ↓
Record diagnosis date
      ↓
Record observed symptoms
      ↓
Select treatedBy
      ↓
Add medicine(s)
      ↓
Enter dosage/frequency/duration
      ↓
Set estimated cost
      ↓
Set follow-up date
      ↓
Create Treatment
      ↓
Status = Ongoing
```

---

# 66. Treatment Completion Workflow

```text
Ongoing Treatment
       ↓
Medicine course completed
       ↓
Follow-up / reassessment
       ↓
Animal recovered
       ↓
completedDate
       ↓
status = Recovered
       ↓
Update animal healthStatus
       ↓
AuditLog
```

---

# 67. Treatment Referral Workflow

```text
Ongoing
   ↓
Case requires higher-level care
   ↓
status = Referred
   ↓
remarks = referral information
   ↓
Notification / follow-up
   ↓
AuditLog
```

---

# 68. Treatment Cancellation Workflow

```text
Ongoing
   ↓
Cancel treatment
   ↓
Record reason in remarks
   ↓
status = Cancelled
   ↓
Reverse/adjust any reserved or consumed inventory where applicable
   ↓
AuditLog
```

---

# 69. Animal Health Status Integration

Animal has:

```text
healthStatus:
Healthy | Sick | Under Treatment | Quarantine
```

Therefore Treatment can update the animal's health state.

Example:

```text
Treatment created
      ↓
Animal.healthStatus = Under Treatment
```

After recovery:

```text
Treatment.status = Recovered
      ↓
Animal.healthStatus = Healthy
```

But the system should not blindly overwrite health status if another active disease/treatment still exists.

---

# 70. Multiple Active Treatments

Suppose:

```text
Animal A
Treatment 1 = Ongoing
Treatment 2 = Ongoing
```

If Treatment 1 becomes:

```text
Recovered
```

the system should NOT automatically set:

```text
Animal.healthStatus = Healthy
```

if Treatment 2 is still active.

Better logic:

```text
Any active treatment?
    ↓ Yes
Under Treatment

No active treatment
    ↓
Healthy / other appropriate state
```

This is important.

---

# 71. Treatment + Quarantine

If the animal is:

```text
healthStatus = Quarantine
```

Treatment can still be recorded.

Quarantine is a farm-management/health status, while Treatment is the clinical event.

Do not automatically equate:

```text
Quarantine = Treatment
```

---

# 72. Treatment + Medicine Inventory

When medicines are actually consumed:

```text
Treatment
   ↓
Medicine
   ↓
Inventory
   ↓
Inventory_Transaction
```

The finalized Inventory_Transaction model supports:

```text
transactionType = Treatment Usage
referenceType = Treatment
referenceId = Treatment._id
```

fileciteturn14file16

---

# 73. When to Consume Medicine Inventory

Do not automatically reduce stock merely because:

```text
Treatment = Ongoing
```

A clean implementation should consume stock when the medicine is actually administered/recorded according to the treatment workflow.

The exact current Treatment model does not contain a separate per-dose administration collection.

Therefore, the implementation must define whether consumption occurs:

```text
per completed/recorded treatment
```

or through a future dose-administration mechanism.

For exact stock accounting, a future `TreatmentAdministration`/dose-log collection could be considered, but it is **not part of the current finalized model**.

---

# 74. Atomic Treatment + Inventory Update

When treatment consumption is recorded:

```text
BEGIN TRANSACTION

Validate Treatment
Validate Medicine
Validate Inventory
Record treatment/administration
Reduce inventory
Create Inventory_Transaction
Create AuditLog

COMMIT
```

If a critical operation fails:

```text
ROLLBACK
```

This prevents health records and inventory from becoming inconsistent.

---

# 75. Insufficient Medicine Stock

Suppose:

```text
Required medicine:
10 ml

Available:
5 ml
```

Recommended:

```text
Block the consumption/completion action
```

unless an authorized manual adjustment/override is provided.

The system should not silently allow:

```text
Stock = -5
```

---

# 76. Expired Medicine

Before medicine use:

```text
Inventory.expiryDate
```

should be checked.

If stock is:

```text
Expired
```

it should not be used for a normal treatment workflow.

The medicine master can remain active while a particular inventory batch is expired.

---

# 77. Prescription-Required Medicine

If:

```text
Medicine.prescriptionRequired = true
```

the treatment workflow can require appropriate authorization.

Since the current architecture does not have a Doctor model:

```text
treatedBy
```

should use the authorized user/farm-member identity according to the project's permission system.

---

# 78. Treatment + Withdrawal Period

Each Medicine may contain:

```text
withdrawalPeriodDays
```

Treatment identifies:

```text
which animal
+
which medicine
+
when
```

Therefore:

```text
Treatment
   ↓
Medicine.withdrawalPeriodDays
   ↓
Last applicable administration
   ↓
Withdrawal end
```

This can support:

```text
Milk warning
Animal sale warning
Food-product safety warning
```

---

# 79. Multiple Medicines with Different Withdrawal Periods

Example:

```text
Medicine A:
withdrawal = 5 days

Medicine B:
withdrawal = 10 days
```

Same Treatment:

```text
Medicine A + B
```

The animal-level withdrawal period should be based on the applicable product/route requirements.

A conservative implementation can use:

```text
maximum applicable withdrawal end date
```

but the exact business rule should follow the approved veterinary/product policy.

The current Treatment model does not contain a dedicated `withdrawalEndDate` field.

---

# 80. Important Limitation

Because the current Treatment model stores only:

```text
diagnosisDate
medicines[]
followUpDate
completedDate
```

and not individual administration timestamps, precise per-dose withdrawal calculation may be limited.

For the current project, use the treatment schedule/recorded dates consistently.

If precise regulatory-grade withdrawal tracking becomes a requirement, a future dose-administration/history model would be a stronger design.

---

# 81. Treatment + Disease Symptoms

Disease master:

```text
Disease.symptoms[]
```

Treatment:

```text
Treatment.symptoms[]
```

The system can show:

```text
Expected/general symptoms
+
Observed symptoms
```

This is useful for health history.

---

# 82. Treatment + Vaccination

Both belong to the Health module but represent different events:

```text
Vaccination
=
Preventive

Treatment
=
Curative/management
```

Animal timeline can combine both:

```text
Aug 10 → Treatment
Aug 17 → Vaccination
Aug 22 → Follow-up
```

---

# 83. Treatment + Disease Master

When a Disease record is deactivated:

```text
Existing Treatment
   ↓
still valid
```

because historical records should remain readable.

The Disease master should not be deleted merely because it is no longer used.

---

# 84. Treatment + Animal Life Status

New Treatment should normally be blocked when:

```text
Animal.lifeStatus = Dead
```

or:

```text
Sold
```

Historical Treatment records remain.

If:

```text
Transferred
```

the farm context should be handled carefully so the new farm does not incorrectly modify old treatment history.

---

# 85. Treatment + Reproduction

Some treatment may be associated with reproductive health.

However:

```text
Treatment
=
health treatment event

Breeding
=
breeding event

Pregnancy
=
pregnancy state/event
```

They should remain separate collections.

Treatment can appear in the animal health timeline alongside reproduction records.

---

# 86. Treatment + Notification

Potential notifications:

```text
Follow-up reminder
Treatment overdue/pending follow-up
Medicine reminder
Referral follow-up
Treatment completed
```

The current Notification model supports:

```text
referenceType = Treatment
referenceId
```

so notifications can directly point back to a Treatment record. fileciteturn14file7

---

# 87. Treatment + Report

The finalized Report model supports:

```text
reportType = Treatment
```

Therefore Liventra can generate:

```text
Treatment History Report
Disease Treatment Report
Medicine Usage Report
Treatment Cost Report
Recovery Report
```

The Report model supports PDF, Excel, CSV and JSON formats. fileciteturn14file7

---

# 88. Treatment Analytics

Possible metrics:

```text
Total treatments
Ongoing cases
Recovered cases
Referred cases
Cancelled cases
Most common diseases
Most used medicines
Average treatment cost
Average recovery duration
Treatment count by month
```

These are derived analytics from:

```text
Treatment
+
Disease
+
Medicine
```

---

# 89. Average Recovery Duration

For recovered treatments:

```text
Recovery Duration
=
completedDate - diagnosisDate
```

Example:

```text
Diagnosis:
Aug 17

Completed:
Aug 22

Duration:
5 days
```

This can be used for reporting and historical analysis.

---

# 90. Treatment Cost Analytics

`estimatedCost` can be aggregated:

```text
Monthly estimated treatment cost
Disease-wise treatment cost
Animal-wise treatment cost
```

But this should not be confused with actual financial expenses.

Actual financial records belong to:

```text
Expense
```

and inventory transactions.

---

# 91. Treatment + Expense

A treatment may cause financial activity:

```text
Treatment
   ↓
Medicine usage
   ↓
Inventory Transaction
   ↓
Expense / purchase cost
```

The Treatment's:

```text
estimatedCost
```

should not automatically create an Expense unless the financial workflow explicitly defines that behavior.

---

# 92. Treatment + AuditLog

Important events:

```text
Treatment Created
Treatment Updated
Treatment Started/Ongoing
Treatment Recovered
Treatment Referred
Treatment Cancelled
Medicine Added
Medicine Removed
Dosage Changed
Frequency Changed
Follow-up Changed
Completed Date Changed
```

These should be auditable.

The finalized AuditLog supports:

```text
entityType = Treatment
action = Create | Update | Delete | Archive | Approve | Reject
```

along with old/new data and changed fields. fileciteturn14file10

---

# 93. Audit for Medicine Changes

Suppose:

```text
Treatment:
Medicine A
5 ml
```

is changed to:

```text
Medicine A
10 ml
```

Audit should capture:

```text
oldData
newData
changedFields
```

This is especially important for health records.

---

# 94. Treatment History Should Be Append-Friendly

Health records are historical.

Therefore:

```text
Create
+
Controlled Update
+
Audit
```

is preferable to destructive deletion.

A completed/recovered treatment should not be silently deleted.

---

# 95. Correction Workflow

If the wrong medicine was entered:

```text
Wrong record
   ↓
Authorized correction
   ↓
Update Treatment
   ↓
Inventory reconciliation if necessary
   ↓
AuditLog
```

If the wrong animal was entered and downstream inventory/history is already affected, a controlled correction/void workflow is safer than simple deletion.

---

# 96. Status + Animal Health Logic

Recommended:

### Treatment created:

```text
status = Ongoing
```

Animal:

```text
healthStatus = Under Treatment
```

### Treatment recovered:

```text
status = Recovered
```

Then check:

```text
Any other active treatment?
```

If no:

```text
Animal.healthStatus = Healthy
```

Otherwise:

```text
Animal.healthStatus = Under Treatment
```

---

# 97. Referral Logic

When:

```text
status = Referred
```

the animal may still be:

```text
Sick
```

or:

```text
Under Treatment
```

depending on the case.

The system should not automatically mark the animal Healthy.

---

# 98. Cancellation Logic

When:

```text
status = Cancelled
```

the system should determine whether:

```text
medicine was already consumed
```

If yes:

```text
Inventory reconciliation
```

may be necessary.

If no:

```text
No medicine consumption
```

should remain.

---

# 99. Duplicate Treatment Prevention

Potential duplicate:

```text
Same farm
Same animal
Same disease
Same diagnosis date
Same medicines
```

The system should warn about a possible duplicate.

But do not enforce a permanent uniqueness constraint on those fields because:

```text
The same animal can legitimately have the same disease multiple times.
```

---

# 100. Multiple Treatment Episodes

Example:

```text
Animal A
FMD
Aug 1 → Recovered

Animal A
FMD
Oct 10 → Ongoing
```

These are two legitimate treatment episodes.

Therefore, Disease + Animal is not a unique pair.

---

# 101. Treatment Number as Episode Identifier

Each treatment episode gets its own:

```text
treatmentNo
```

Example:

```text
TRT-000001
TRT-000015
```

This preserves separate treatment episodes.

---

# 102. Search and Filtering

Treatment list should support filtering by:

```text
treatmentNo
farm
animal
disease
treatedBy
status
diagnosis date
follow-up date
completed date
```

Example:

```text
Status = Ongoing
```

shows active treatment cases.

---

# 103. Animal Treatment History Query

Conceptually:

```text
Find Treatment
where:
animalId = selectedAnimal
sort:
diagnosisDate descending
```

Result:

```text
Latest treatment first
```

This supports the animal health history screen.

---

# 104. Ongoing Treatment Dashboard

Farm dashboard can show:

```text
Ongoing Treatments
Follow-ups Due
Recovered Cases
Referred Cases
Cancelled Cases
```

This directly supports the SRS health-management requirements.

---

# 105. Follow-up Dashboard

Query:

```text
status = Ongoing
AND
followUpDate <= reminder threshold
```

Then:

```text
Create Notification
```

This makes treatment follow-up proactive rather than passive.

---

# 106. Treatment Creation Authorization

Recommended:

```text
Admin
Farm Owner
Farm Manager
Authorized Worker
```

depending on final permission matrix.

A user must belong to the farm before creating Treatment.

The SRS defines:

```text
Admin = Full Access
Farm Owner = Manage Everything
Farm Manager = Regular Management
Worker = Data Entry
```

so Treatment permissions should follow the final role-permission matrix. fileciteturn14file19

---

# 107. `treatedBy` Validation

The selected `treatedBy` user should:

```text
exist
+
be associated with the farm
+
have permission to record/perform treatment
```

This prevents:

```text
Farm A treatment
treatedBy = unrelated user
```

---

# 108. Medicine Authorization

If any selected Medicine has:

```text
prescriptionRequired = true
```

the Treatment workflow should apply the appropriate authorization rule.

The current architecture does not require a separate Doctor model.

---

# 109. Treatment + Inventory Transaction Example

Suppose:

```text
Treatment:
TRT-000001

Medicine:
MED-001

Quantity consumed:
5 ml
```

Create:

```text
Inventory_Transaction
transactionType = Treatment Usage
referenceType = Treatment
referenceId = TRT-000001
quantity = 5
unit = ml
performedBy = user
```

This gives traceability.

---

# 110. Important Unit Consistency

Medicine/Treatment may use:

```text
ml
mg
tablet
dose
```

Inventory may use:

```text
kg
g
liter
ml
piece
box
packet
bottle
```

Therefore, consumption must use compatible units.

Example:

```text
Treatment:
5 ml

Inventory:
Bottle
```

The system may need a conversion/pack-size rule if inventory is stored as bottles but treatment consumes ml.

The current model does not define a dedicated conversion field.

---

# 111. Current Model Limitation — Inventory Pack Conversion

Example:

```text
Inventory:
10 bottles

Each bottle:
100 ml

Treatment:
5 ml
```

The current Inventory model does not explicitly store:

```text
packSize = 100 ml
```

Therefore, exact fractional bottle consumption needs either:

```text
inventory unit = ml
```

or a future pack-size/conversion mechanism.

This is an implementation consideration, not a silent schema change.

---

# 112. Current Model Limitation — Per-Dose Administration

The current Treatment model stores:

```text
frequency
duration
```

but not:

```text
individual administration timestamps
```

Therefore, if Liventra needs:

```text
Dose 1 at 08:00
Dose 2 at 20:00
Dose 3 missed
```

a future administration-log model would be more precise.

For the current project, the finalized Treatment model should remain unchanged.

---

# 113. Current Model Limitation — Referral Details

Current status includes:

```text
Referred
```

but no dedicated fields such as:

```text
referredTo
referralDate
referralReason
```

Therefore, current implementation should use:

```text
remarks
```

for referral information.

If referral management becomes a major feature, a future schema enhancement may be appropriate.

---

# 114. Current Model Limitation — Diagnosis Confidence

Current model does not contain:

```text
diagnosisConfidence
diagnosisMethod
laboratoryTest
```

Therefore, do not invent those fields.

The current Treatment model simply records:

```text
diseaseId
symptoms
diagnosisDate
```

---

# 115. Current Model Limitation — Doctor Information

There is no Doctor model in the current architecture.

Therefore:

```text
treatedBy
```

should reference the existing authorized user/farm-member identity.

Do not reintroduce:

```text
doctorId
```

unless the project explicitly adds a Doctor collection later.

---

# 116. Recommended Service-Layer Methods

A clean backend can use:

```text
treatmentService.create()
treatmentService.update()
treatmentService.complete()
treatmentService.refer()
treatmentService.cancel()
treatmentService.addMedicine()
treatmentService.removeMedicine()
treatmentService.scheduleFollowUp()
treatmentService.calculateWithdrawal()
treatmentService.consumeMedicine()
```

---

# 117. Why Service Layer Is Important

Treatment completion may affect multiple collections:

```text
Treatment
Animal
Inventory
Inventory_Transaction
Notification
AuditLog
```

Therefore, a controller containing all business logic would become difficult to maintain.

A service layer provides a cleaner architecture.

---

# 118. Transaction Boundary

For a critical treatment operation:

```text
BEGIN TRANSACTION

Validate Treatment
Validate Animal
Validate Disease
Validate Medicine
Validate Inventory
Update Treatment
Update Animal health status
Update Inventory
Create Inventory_Transaction
Create AuditLog

COMMIT
```

If a critical operation fails:

```text
ROLLBACK
```

This prevents partial updates.

---

# 119. Notification Timing

Do not create notifications blindly on every Treatment update.

Useful triggers:

```text
Treatment created
Follow-up approaching
Follow-up overdue
Treatment recovered
Treatment referred
```

Avoid duplicate notifications by using:

```text
referenceType = Treatment
referenceId = Treatment._id
```

and appropriate notification-state checks.

---

# 120. Treatment + Health History

The SRS includes:

```text
Health History
```

as a health module requirement. fileciteturn14file19

Treatment is one of the main sources for this history.

Example:

```text
Animal A Health History

Aug 10
Treatment — FMD

Aug 15
Follow-up

Aug 17
Recovered

Aug 20
Vaccination
```

---

# 121. Treatment + Reports

The SRS/report architecture supports Treatment reporting.

Possible report:

```text
Treatment Report
```

with filters:

```text
Date range
Animal
Disease
Status
Farm
```

Output:

```text
PDF
Excel
CSV
JSON
```

as supported by the finalized Report model. fileciteturn14file7

---

# 122. Example — Complete Treatment

```javascript
{
    treatmentNo: "TRT-000001",

    farmId: "FARM-001",

    animalId: "AN-000125",

    diseaseId: "DIS-FMD-001",

    diagnosisDate: "2026-08-17",

    symptoms: [
        "Fever",
        "Reduced appetite"
    ],

    treatedBy: "FARMMEMBER-001",

    medicines: [
        {
            medicineId: "MED-001",
            dosage: 5,
            dosageUnit: "ml",
            frequency: "Once daily",
            duration: "5 days",
            administrationRoute: "IM",
            instructions: "Follow the authorized treatment plan."
        }
    ],

    estimatedCost: 850,

    status: "Ongoing",

    followUpDate: "2026-08-22",

    completedDate: null,

    remarks: "Follow-up required.",

    createdBy: "FARMMEMBER-002",

    createdAt: "2026-08-17T10:00:00Z",

    updatedAt: "2026-08-17T10:00:00Z"
}
```

Values are illustrative only.

---

# 123. Example — Recovered Treatment

```javascript
{
    treatmentNo: "TRT-000001",

    status: "Recovered",

    completedDate: "2026-08-22",

    remarks: "Animal recovered after treatment."
}
```

The system should then check whether any other active treatment exists before setting the Animal health status to Healthy.

---

# 124. Example — Referred Treatment

```javascript
{
    treatmentNo: "TRT-000002",

    status: "Referred",

    remarks:
        "Case referred for specialized veterinary evaluation."
}
```

The current schema has no dedicated referral fields.

---

# 125. Example — Cancelled Treatment

```javascript
{
    treatmentNo: "TRT-000003",

    status: "Cancelled",

    remarks:
        "Cancelled due to duplicate treatment entry."
}
```

If medicine inventory was already consumed, reconciliation is required.

---

# 126. Error Scenario — Animal From Another Farm

Input:

```text
farmId = FARM-A
animalId = animal belonging to FARM-B
```

Backend:

```text
Reject
```

---

# 127. Error Scenario — Disease Does Not Exist

Input:

```text
diseaseId = invalid
```

Backend:

```text
Reject
```

---

# 128. Error Scenario — Medicine Does Not Exist

Input:

```text
medicines[].medicineId = invalid
```

Backend:

```text
Reject
```

---

# 129. Error Scenario — Inactive Medicine

Input:

```text
Medicine.isActive = false
```

Recommended:

```text
Reject for new routine treatment
```

Historical treatment remains valid.

---

# 130. Error Scenario — Invalid Dosage

```text
dosage = -5
```

Reject.

---

# 131. Error Scenario — Invalid Route

```text
Medicine:
Ointment

Treatment:
IV
```

Reject.

---

# 132. Error Scenario — Recovered Without Completed Date

Input:

```text
status = Recovered
completedDate = null
```

Recommended:

```text
Validation error
```

---

# 133. Error Scenario — Ongoing With Completed Date

Input:

```text
status = Ongoing
completedDate = 2026-08-22
```

Recommended:

```text
Reject or clear completedDate
```

because the status is still ongoing.

---

# 134. Error Scenario — Follow-up Before Diagnosis

Input:

```text
diagnosisDate = Aug 20
followUpDate = Aug 18
```

Reject unless explicitly recorded as a historical correction.

---

# 135. Error Scenario — Duplicate Treatment

Potential duplicate:

```text
Same animal
Same disease
Same diagnosis date
Same medicine
```

System should warn.

But do not permanently forbid it because repeated disease episodes are legitimate.

---

# 136. Error Scenario — Treatment Completed With Insufficient Stock

If completion requires medicine consumption:

```text
Available stock < required quantity
```

then:

```text
Do not complete/consume automatically
```

unless an authorized exception exists.

---

# 137. Final Treatment Lifecycle

```text
                    ┌──────────────┐
                    │   Created    │
                    └──────┬───────┘
                           ↓
                    ┌──────────────┐
                    │   Ongoing    │
                    └───┬──────┬───┘
                        │      │
                        │      └─────────────┐
                        ↓                    ↓
                 ┌─────────────┐      ┌─────────────┐
                 │  Recovered  │      │   Referred  │
                 └─────────────┘      └─────────────┘

                        │
                        ↓
                  ┌─────────────┐
                  │  Cancelled  │
                  └─────────────┘
```

`Cancelled` should normally be a controlled terminal state for that treatment episode.

---

# 138. Final Treatment Architecture

```text
                 ┌──────────────┐
                 │    Animal    │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │   Treatment  │
                 └──────┬───────┘
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
       Disease       Medicine      Follow-up
          │             │             │
          │             ▼             ▼
          │         Inventory     Notification
          │             │
          │             ▼
          │    Inventory_Transaction
          │
          ▼
       Health History
```

---

# 139. Final Business Rules Checklist

```text
[✓] Treatment is an animal-specific health event
[✓] Disease is master data
[✓] Medicine is master data
[✓] treatmentNo should be unique
[✓] farmId is required
[✓] animalId is required
[✓] Animal must belong to farmId
[✓] diseaseId references Disease._id
[✓] Current finalized model uses singular diseaseId
[✓] diagnosisDate records actual diagnosis/record date
[✓] symptoms[] stores observed symptoms
[✓] Disease.symptoms[] remains general/reference information
[✓] treatedBy references authorized farm/user identity
[✓] No Doctor model is required in current architecture
[✓] createdBy may differ from treatedBy
[✓] medicines[] supports multiple medicines per treatment
[✓] medicineId references Medicine._id
[✓] dosage is treatment-specific
[✓] dosageUnit must be compatible with medicine
[✓] frequency describes administration frequency
[✓] duration describes treatment duration
[✓] administrationRoute should match medicine compatibility
[✓] instructions are treatment-specific
[✓] estimatedCost is not the same as actual Expense
[✓] status values are Ongoing, Recovered, Referred, Cancelled
[✓] Recovered should normally require completedDate
[✓] Ongoing normally should not have completedDate
[✓] followUpDate can trigger Notification
[✓] Medicine withdrawal rules can support milk/product warnings
[✓] Medicine inventory should not be consumed merely because a treatment exists
[✓] Treatment Usage can create Inventory_Transaction
[✓] Insufficient stock should normally block consumption
[✓] Expired medicine stock should not be used
[✓] Treatment can update Animal.healthStatus
[✓] Recovery must check for other active treatments
[✓] Historical treatment records should not be casually deleted
[✓] Important changes should generate AuditLog
[✓] Treatment supports Health History
[✓] Treatment supports Treatment Reports
[✓] Duplicate episodes of the same disease are legitimate
[✓] Duplicate accidental records should be detected/warned
[✓] Multi-collection operations should use service/transaction logic
[✓] Current model does not include per-dose administration history
[✓] Current model does not include dedicated referral fields
[✓] Current model does not include a Doctor collection
```

---

# 140. Final Status

**Model:** `Treatment`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Disease Integration:** ✅  
**Medicine Integration:** ✅  
**Animal Health Integration:** ✅  
**Inventory Integration:** ✅  
**Withdrawal Support:** ✅  
**Follow-up/Notification:** ✅  
**Health History:** ✅  
**Reporting:** ✅  
**Auditability:** ✅  
**Historical Integrity:** ✅  

**Next Model:** `Breeding`
